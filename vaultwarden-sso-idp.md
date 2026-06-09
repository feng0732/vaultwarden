# Vaultwarden SSO 登录与 IdP 集成代码分析

## 一、整体架构概览

Vaultwarden 的 SSO 功能基于 **OpenID Connect (OIDC)** 协议实现，涉及三个核心模块：

| 模块 | 文件 | 职责 |
|------|------|------|
| SSO 核心逻辑 | [sso.rs](src/sso.rs) | 授权 URL 构建、Code 交换、Token 管理、账号兑换 |
| OIDC 客户端 | [sso_client.rs](src/sso_client.rs) | 与 IdP 通信：Discovery、Token 端点、UserInfo 端点 |
| API 路由层 | [identity.rs](src/api/identity.rs) | HTTP 端点：`/connect/authorize`、`/connect/oidc-signin`、`/connect/token` |
| 数据模型 | [sso_auth.rs](src/db/models/sso_auth.rs)、[user.rs](src/db/models/user.rs) | `sso_auth` 临时会话表、`sso_users` 账号映射表 |
| 配置 | [config.rs](src/config.rs) | SSO 相关环境变量解析与校验 |

数据模型关系图：

```
sso_auth (临时会话, 10分钟过期)
├── state (PK, OIDCState)
├── client_challenge (PKCE)
├── nonce
├── redirect_uri
├── code_response / code_response_error
├── auth_response (OIDCAuthenticatedUser JSON)
└── binding_hash (浏览器绑定 Cookie 哈希)

sso_users (永久账号映射)
├── user_uuid (PK, FK -> users.uuid)
└── identifier (格式: "{issuer}/{subject}")

users
└── uuid (PK)
```

---

## 二、身份提供方（IdP）配置

### 2.1 核心配置项

所有配置在 [config.rs](src/config.rs#L799-L832) 的 `sso` 块中定义：

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `sso_enabled` | bool | false | 启用 SSO |
| `sso_only` | bool | false | 禁用邮箱+主密码登录，仅允许 SSO |
| `sso_client_id` | String | - | OIDC Client ID (必填) |
| `sso_client_secret` | Pass | - | OIDC Client Secret (必填) |
| `sso_authority` | String | - | IdP 根 URL，不含 `/.well-known/openid-configuration` (必填) |
| `sso_scopes` | String | "email profile" | 请求的 Scope，`openid` 为隐式包含 |
| `sso_authorize_extra_params` | String | - | 授权请求额外参数，URL query 格式 |
| `sso_pkce` | bool | true | 启用 PKCE |
| `sso_signups_match_email` | bool | true | 允许通过邮箱关联已有非 SSO 用户 |
| `sso_allow_unknown_email_verification` | bool | false | 允许 IdP 不返回 email_verified 的情况 |
| `sso_callback_path` | String | generated | 自动生成：`{domain}/identity/connect/oidc-signin` |
| `sso_audience_trusted` | String (regex) | - | 额外信任的 Id Token audience (正则) |
| `sso_auth_only_not_session` | bool | false | SSO 仅用于认证，会话生命周期走 Vaultwarden 默认逻辑 |
| `sso_client_cache_expiration` | u64 | 0 | Discovery 客户端缓存秒数，0 禁用 |
| `sso_debug_tokens` | bool | false | Debug 级别打印 Token |

启动校验在 [config.rs](src/config.rs#L1079-L1087)：`sso_enabled=true` 时，`sso_client_id`、`sso_client_secret`、`sso_authority` 三者缺一不可。

### 2.2 OIDC 客户端初始化与缓存

客户端实现在 [sso_client.rs](src/sso_client.rs#L62-L155)：

```rust
// 类型别名：基于 openidconnect crate 的 CoreClient
pub type CustomClient = openidconnect::Client<...>;

pub struct Client {
    pub http_client: OidcHttpClient,  // 封装 reqwest，禁止重定向
    pub core_client: CustomClient,    // openidconnect 官方客户端
}
```

初始化流程（`Client::get_client()` [L101-L140](src/sso_client.rs#L101-L140)）：

1. 从 `CONFIG` 读取 `client_id`、`client_secret`、`issuer_url`
2. 调用 `CoreProviderMetadata::discover_async(issuer_url, ...)` → 请求 IdP 的 `/.well-known/openid-configuration`
3. 从返回的元数据中提取 `token_uri` 和 `user_info_url`（若缺失则报错）
4. 设置 `redirect_uri` → 返回 `Client`

缓存使用 `moka::sync::Cache`，容量为 1，TTL 由 `sso_client_cache_expiration` 控制（`Client::cached()` [L143-L155](src/sso_client.rs#L143-L155)）。

---

## 三、登录回调流程

完整的 OIDC Authorization Code Flow 涉及 5 个关键 HTTP 端点：

```
┌──────────────┐     1. /sso/prevalidate       ┌──────────────────┐
│   Client     │ ──────────────────────────────▶│   Vaultwarden    │
│ (Web/App)    │ ◀──────────────────────────────│   /identity      │
└──────┬───────┘     返回 sso_token             └────────┬─────────┘
       │                                                │
       │     2. /connect/authorize                      │
       │  (client_id, redirect_uri, state,              │
       │   code_challenge, code_challenge_method=S256)  │
       ├───────────────────────────────────────────────▶│
       │                                                │ 生成 SSO_BINDING cookie
       │                                                │ 构建 sso_auth 记录
       │     3. 302 重定向到 IdP 授权页                  │
       │◀───────────────────────────────────────────────┤
       │                                                │
       │                                                │
       │     4. 用户在 IdP 登录并授权                    │
       │ ──────────────────────────────┐                │
       │                               ▼                │
       │                          ┌─────────┐           │
       │                          │   IdP   │           │
       │                          └────┬────┘           │
       │                               │                │
       │     5. IdP 回调 Vaultwarden   │                │
       │        /connect/oidc-signin   │                │
       │        ?code=...&state=...    │                │
       │◀──────────────────────────────┤                │
       │                               │                │
       │     6. 302 重定向回 Client    │                │
       │        (redirect_uri + code + state)           │
       ├───────────────────────────────────────────────▶│
       │                                                │
       │     7. /connect/token                          │
       │        grant_type=authorization_code           │
       │        code=...&code_verifier=...              │
       ├───────────────────────────────────────────────▶│
       │                                                │ 交换 Code → Token
       │                                                │ UserInfo 请求
       │                                                │ 账号映射 / 注册
       │     8. 返回 access_token + refresh_token       │ 2FA 校验
       │◀───────────────────────────────────────────────┘
```

### 3.1 端点 1：`/sso/prevalidate` — 预校验

实现在 [identity.rs](src/api/identity.rs#L1155-L1165)。仅检查 `sso_enabled`，返回一个短期（2 分钟）JWT，issuer 为 `{domain_origin}|sso`。

### 3.2 端点 2：`/connect/authorize` — 发起授权

实现在 [identity.rs](src/api/identity.rs#L1271-L1305)。

**输入参数**（`AuthorizeData`）：`client_id`、`redirect_uri`、`state`、`code_challenge`、`code_challenge_method`（必须为 `S256`）。

**核心逻辑**：

1. **浏览器绑定保护**：生成 32 字节随机 `binding_token`，计算 SHA-256 得到 `binding_hash` 存入 DB；明文值通过 `VW_SSO_BINDING` Cookie 返回浏览器（HttpOnly、SameSite=Lax、Path=`/identity/connect/`）。
2. 调用 [sso::authorize_url()](src/sso.rs#L188-L215) 构建 IdP 授权 URL。
3. 生成 `SsoAuth` 记录并保存到 DB。

`redirect_uri` 的转换规则（`sso::authorize_url` [L196-L210](src/sso.rs#L196-L210)）：

| client_id | 重定向目标 |
|-----------|------------|
| web / browser | `{domain}/sso-connector.html` |
| desktop / mobile | `bitwarden://sso-callback` |
| cli | `http://localhost:{port}` (从原始 redirect_uri 中提取端口) |

### 3.3 端点 3：`/connect/oidc-signin` — IdP 回调

两个重载路由（[identity.rs L1169-L1244](src/api/identity.rs#L1169-L1244)）：

- 成功回调：`?code=&state=` → [oidcsignin()](src/api/identity.rs#L1169-L1172)
- 错误回调：`?state=&error=&error_description=` → [oidcsignin_error()](src/api/identity.rs#L1177-L1196)

两者均调用内部函数 [oidcsignin_redirect()](src/api/identity.rs#L1201-L1244)：

1. Base64 解码 `state` → 从 DB 查找 `SsoAuth` 记录（10 分钟内有效）。
2. **浏览器绑定校验**：读取 `VW_SSO_BINDING` Cookie，计算 SHA-256 与 `sso_auth.binding_hash` 比较，不一致则拒绝。
3. 将 `code`（或错误信息）写入 `sso_auth.code_response` / `sso_auth.code_response_error`。
4. 重定向回 Client 的 `redirect_uri`，附加参数：`code`、`state`、`scope=api`、`iss={domain}`。

> **注意**：错误回调时，`code` 被替换为 `state` 的值（编码后的），这样后续 Client 用这个"code"请求 Token 时，Vaultwarden 可以从 DB 中检索到错误信息并报告。

### 3.4 端点 4：`/connect/token` (grant_type=authorization_code) — 兑换 Token

入口在 [identity.rs L99-L109](src/api/identity.rs#L99-L109)，实际处理在 [sso_login()](src/api/identity.rs#L178-L346)。

此函数完成以下工作：

1. **Code 交换**：调用 [sso::exchange_code()](src/sso.rs#L247-L316)
   - 从 DB 检索 `SsoAuth`
   - 若已有 `auth_response`（2FA 重试场景）直接返回
   - 若存在 `code_response_error` 则报错并删除记录
   - 调用 [Client::exchange_code()](src/sso_client.rs#L194-L254)：
     - 向 IdP Token 端点发送 `authorization_code` + PKCE verifier
     - 验证 `id_token`（nonce、audience、签名、issuer）
     - 返回 `(token_response, id_claims)`
   - 调用 [Client::user_info()](src/sso_client.rs#L256-L261) 获取用户信息
   - 合并邮箱：`id_claims.email() ?? user_info.email()`，转小写
   - 构造 `OIDCAuthenticatedUser`（含 access_token、refresh_token、identifier、email 等）存入 `sso_auth.auth_response`
2. **账号映射**（详见下一章）
3. **2FA 校验**（如有）
4. **Token 兑换**：调用 [sso::redeem()](src/sso.rs#L319-L355)，删除 `sso_auth` 记录，生成 Vaultwarden 的 access/refresh token。

---

## 四、账号映射逻辑

账号映射是 SSO 登录最关键的部分。它在 [sso_login()](src/api/identity.rs#L208-L337) 中执行，**分两级查找**。

### 4.1 映射规则详解

```
OIDCAuthenticatedUser.identifier = "{issuer}/{subject}"
                           email = 从 Id Token / UserInfo 获取
```

**第一级：按 OIDC Identifier 精确匹配**
- 调用 [SsoUser::find_by_identifier()](src/db/models/user.rs#L529-L539)
- 执行 `users INNER JOIN sso_users WHERE sso_users.identifier = ?`
- 命中 → 这是已建立 SSO 关联的用户，直接继续登录流程

**第二级：按 Email 匹配**（仅当第一级未命中）
- 调用 [SsoUser::find_by_mail()](src/db/models/user.rs#L541-L553)
- 执行 `users LEFT JOIN sso_users WHERE users.email = ?`

Email 匹配后再分三种情况：

| 情况 | 条件 | 结果 |
|------|------|------|
| A. 已有 SSO 用户同邮箱 | 返回的 `sso_user` 为 Some | ❌ 登录失败（安全保护：两个不同的 OIDC identifier 不能共享邮箱） |
| B. 已有非 SSO 用户 + 关联被禁用 | `user.private_key.is_some()` 且 `sso_signups_match_email=false` | ❌ 登录失败 |
| C. 已有非 SSO 用户 + 邮箱未验证通过 | `email_verified=None` 且 `sso_allow_unknown_email_verification=false`；或 `email_verified=Some(false)` | ❌ 登录失败 |
| D. 已有非 SSO 用户 + 邮箱验证通过 | 以上都不满足 | ✅ 允许关联，继续登录（后续 redeem() 时创建 sso_users 记录） |

**两级均未命中 → 新用户注册**：
- 校验邮箱域名是否在白名单
- 校验邮箱验证状态（同上 C 条件）
- 创建 `User`（`verified_at` 设为当前时间）
- 创建设备记录
- 不设置主密码（用户首次登录后通过客户端设置）

### 4.2 已有用户的附加处理

当映射到已有用户时（第二级命中 D 或第一级命中）：

1. 检查 `user.enabled`，被禁用则拒绝
2. 走 2FA 流程
3. **Stub 用户完善**：若 `user.private_key.is_none()`（之前被邀请但未完成注册的桩用户），补齐 `verified_at` 和 `name`
4. **邮箱变更通知**：若 IdP 返回的邮箱与 DB 中不一致 → 发送邮件通知旧邮箱

### 4.3 建立 SSO 关联

在登录流程末尾的 [sso::redeem()](src/sso.rs#L319-L355) 中：

```rust
if sso_user.is_none() {
    let user_sso = SsoUser {
        user_uuid: user.uuid.clone(),
        identifier: auth_user.identifier.clone(), // "{issuer}/{subject}"
    };
    user_sso.save(conn).await?;
}
```

- 首次通过 Email 匹配登录的用户会在这里创建 `sso_users` 记录
- 之后再次登录会走第一级 Identifier 精确匹配

### 4.4 Token 生命周期

`sso::redeem()` 根据 `sso_auth_only_not_session` 有两种模式：

| 模式 | access_token 过期 | refresh_token 来源 |
|------|-------------------|-------------------|
| `sso_auth_only_not_session=true` | Vaultwarden 默认（1 小时） | Vaultwarden 默认（30 天），不持有 IdP token |
| `false`（默认） | 优先用 IdP access_token 的 exp，fallback 到 `now + expires_in` | 优先用 IdP refresh_token；若无则回退到用 access_token 代替并定期调用 UserInfo 校验 |

刷新流程在 [sso::exchange_refresh_token()](src/sso.rs#L427-L473)：
- 若保存的是 IdP refresh_token → 调用 IdP Token 端点换新
- 若保存的是 access_token → 调用 UserInfo 端点校验有效性（即将过期时报错）

---

## 五、安全校验深度分析

### 5.1 sso_token 是否被校验

**结论：sso_token 参数完全没有被校验，也没有被使用。**

证据：

1. `AuthorizeData` 结构体在 [identity.rs](src/api/identity.rs#L1246-L1268) 中定义了该字段，但带有 `#[allow(unused)]` 标记：

```rust
#[allow(unused)]
#[field(name = uncased("ssoToken"))]
sso_token: Option<String>,
```

2. `authorize` 处理函数在解构参数时使用了 `..` 模式（[identity.rs](src/api/identity.rs#L1273-L1280)），显式忽略包括 `sso_token` 在内的多个字段：

```rust
let AuthorizeData {
    client_id,
    redirect_uri,
    state,
    code_challenge,
    code_challenge_method,
    ..  // sso_token、response_type、scope、response_mode、domain_hint 全部被忽略
} = data;
```

3. `/sso/prevalidate` 端点返回的 JWT（`encode_ssotoken_claims` 生成，[sso.rs](src/sso.rs#L109-L130)）仅在客户端侧使用，Vaultwarden 服务端从不解码或验证它。该 JWT 包含 `nbf`、`exp`（2分钟）、`iss`、`sub="vaultwarden"`，但**没有对应的 `decode_ssotoken_claims` 函数**。

这意味着攻击者可以：
- 跳过 `/sso/prevalidate` 直接请求 `/connect/authorize`
- 传入任意 `ssoToken` 值或完全不传该参数，不影响授权流程

### 5.2 `/connect/authorize` 参数校验清单

该端点接收 10 个参数，逐项校验情况如下：

| 参数 | 是否必填 | 校验逻辑 | 代码位置 |
|------|----------|----------|----------|
| `client_id` | ✅ 是 | 白名单匹配：仅允许 `"web"`/`"browser"`/`"desktop"`/`"mobile"`/`"cli"`，其他返回错误 | [sso.rs](src/sso.rs#L196-L210) |
| `redirect_uri` | ✅ 是 | 仅 CLI 模式有效（校验 `http://localhost:{4位端口}` 正则）；其他模式下**完全忽略客户端传入的值**，改用固定硬编码目标 | [sso.rs](src/sso.rs#L196-L210) |
| `state` | ✅ 是 | 作为 `OIDCState` newtype 接收（仅 String 包装），**不校验内容格式、长度或随机性**；后续仅作为 DB 查询主键使用 | [identity.rs](src/api/identity.rs#L1258) |
| `code_challenge` | ✅ 是 | 作为 `OIDCCodeChallenge` newtype 接收（仅 String 包装），**不校验 Base64URL 格式或长度**；原样存入 DB 及传给 IdP | [identity.rs](src/api/identity.rs#L1259) |
| `code_challenge_method` | ✅ 是 | **严格校验**：必须等于 `"S256"`，否则 `err!("Unsupported code challenge method")` | [identity.rs](src/api/identity.rs#L1282-L1284) |
| `ssoToken` | ❌ 否 | `#[allow(unused)]`，完全忽略 | [identity.rs](src/api/identity.rs#L1265-L1267) |
| `response_type` | ❌ 否 | `#[allow(unused)]`，完全忽略 | [identity.rs](src/api/identity.rs#L1254-L1255) |
| `scope` | ❌ 否 | `#[allow(unused)]`，完全忽略（真正传给 IdP 的 scope 来自 `CONFIG.sso_scopes`） | [identity.rs](src/api/identity.rs#L1256-L1257) |
| `response_mode` | ❌ 否 | `#[allow(unused)]`，完全忽略 | [identity.rs](src/api/identity.rs#L1261-L1262) |
| `domain_hint` | ❌ 否 | `#[allow(unused)]`，完全忽略 | [identity.rs](src/api/identity.rs#L1263-L1264) |

补充说明：`/connect/token` 阶段的 SSO scope 有校验（`AuthMethod::Sso.check_scope`），必须精确等于 `"api offline_access"`，见 [auth.rs](src/auth.rs#L1165-L1172) 和 [identity.rs](src/api/identity.rs#L186)。

### 5.3 Id Token 的 audience 信任机制

**默认信任规则**：`openidconnect` crate 的 `CoreIdTokenVerifier` 内置要求 Id Token 的 `aud`（audience）声明**必须包含配置的 `client_id`**（即 `CONFIG.sso_client_id`）。这是 OIDC 规范的标准行为。

**扩展信任规则**：Vaultwarden 允许通过 `SSO_AUDIENCE_TRUSTED` 环境变量配置一个**正则表达式**，额外信任匹配该正则的任意 audience。实现在 [sso_client.rs](src/sso_client.rs#L273-L286)：

```rust
pub fn vw_id_token_verifier(&self) -> CoreIdTokenVerifier<'_> {
    let mut verifier = self.core_client.id_token_verifier();
    if let Some(regex_str) = CONFIG.sso_audience_trusted() {
        match Regex::new(&regex_str) {
            Ok(regex) => {
                verifier = verifier.set_other_audience_verifier_fn(move |aud| regex.is_match(aud));
            }
            Err(err) => {
                error!("Failed to parse SSO_AUDIENCE_TRUSTED={regex_str} regex: {err}");
                // 正则编译失败 → 不设置扩展规则，仅使用默认 client_id 校验
            }
        }
    }
    verifier
}
```

验证时机：在 `Client::exchange_code()` 中成功从 IdP 获取 token_response 后立即执行（[sso_client.rs](src/sso_client.rs#L243-L249)）：

```rust
let id_claims = match id_token.claims(&self.vw_id_token_verifier(), &oidc_nonce) {
    Ok(claims) => claims.clone(),
    Err(err) => {
        Self::invalidate();  // 验证失败顺便清空 discovery 缓存
        err!(format!("Could not read id_token claims, {err}"));
    }
};
```

安全提示：`set_other_audience_verifier_fn` 的语义是**在默认 client_id 校验之外附加信任**，不会覆盖内置校验。因此最终判定逻辑是：
- `aud` 包含 `sso_client_id` ✅ 通过
- **或** `aud` 中任意值匹配 `SSO_AUDIENCE_TRUSTED` 正则 ✅ 通过
- 否则 ❌ 拒绝

### 5.4 Discovery 客户端缓存的失效机制

缓存载体：`moka::sync::Cache<String, Client>`，容量为 1，key 固定为 `"sso-client"`，见 [sso_client.rs](src/sso_client.rs#L28-L34)。

**失效方式一：TTL 自然过期**
- `time_to_live(Duration::from_secs(CONFIG.sso_client_cache_expiration()))`
- 默认 `sso_client_cache_expiration = 0`，即缓存**禁用**，每次请求都会重新 Discovery
- 设为正值后，按秒数自然过期

**失效方式二：主动失效（`Client::invalidate()`）**
- 仅在一处触发：当 `exchange_code()` 中 Id Token claims 验证失败时（[sso_client.rs](src/sso_client.rs#L243-L247)）
- 假设场景：IdP 轮换了签名密钥，但缓存的 `CoreClient` 还持有旧的 JWKS → Id Token 签名校验失败 → 清空缓存 → 下次请求重新 Discovery 拉取新 JWKS

**不触发失效的场景（潜在问题点）**：
- UserInfo 端点调用失败 → 不清空缓存（[sso_client.rs](src/sso_client.rs#L263-L270) 只返回错误）
- Refresh Token 交换失败 → 不清空缓存（[sso_client.rs](src/sso_client.rs#L297-L310) 只返回错误）
- Discovery 时无法获取 `token_uri` 或 `user_info_url` → 直接报错不写缓存，不存在失效
- 配置热更新（若 `sso_client_id`/`sso_authority` 变更）→ 缓存不会自动感知（`LazyLock` 在首次调用时读取配置，之后不会重新读取）

失效实现代码（[sso_client.rs](src/sso_client.rs#L157-L161)）：

```rust
pub fn invalidate() {
    if CONFIG.sso_client_cache_expiration() > 0 {
        CLIENT_CACHE.invalidate(&*CLIENT_CACHE_KEY);
    }
    // 缓存禁用时此函数为 no-op
}
```

---

## 六、忽略 sso_token 与 authorize 参数后的安全影响边界

本章的核心问题：既然 `sso_token` 完全未校验，且 `/connect/authorize` 中 6/10 的参数被 `#[allow(unused)]` 直接忽略，那么攻击者能走多远？后续的五道防线（IdP 登录、PKCE、浏览器绑定 Cookie、Id Token 校验、账号映射）各自能拦住什么、拦不住什么？

### 6.1 攻击者能直接跳过的环节

由于 Vaultwarden 服务端对以下内容不做任何校验：

| 被忽略项 | 攻击者可做的操作 |
|----------|-----------------|
| `sso_token` | 完全跳过 `/sso/prevalidate`，直接请求 `/connect/authorize`；传任意值或不传都不影响 |
| `response_type` | 不传或传任意值（IdP 侧仍会校验，但 Vaultwarden 不关心） |
| `scope`（authorize 阶段） | 不传或传任意值；真正传给 IdP 的 scope 来自 `CONFIG.sso_scopes`（默认 `"email profile"`，`openid` 隐式） |
| `response_mode` | 不传或传任意值 |
| `domain_hint` | 不传或传任意值 |
| `state` | 传任意字符串；虽然是必填的 newtype，但仅作为 DB 主键，不校验长度/格式/随机性（极端情况可被暴力猜解） |
| `code_challenge` | 传任意字符串；虽然是必填的 newtype，但不校验 Base64URL 格式或长度 |

此外 `redirect_uri` 在 web/browser/desktop/mobile 模式下也被忽略（直接用硬编码值），仅 CLI 模式会提取端口号。

**注意**：`client_id` 和 `code_challenge_method` 不在此列——前者有白名单校验，后者必须为 `"S256"`。

### 6.2 防线一：IdP 登录（用户在 IdP 侧输入凭据）

**防线位置**：完全发生在 IdP 侧，Vaultwarden 仅重定向引导。

**能拦住**：
- ❌ 没有 IdP 合法账号凭据的攻击者（即使绕过 Vaultwarden 的所有前置检查，IdP 登录页会挡住）
- ❌ 想冒用他人 IdP 账号但无法通过其 MFA/密码的攻击者

**拦不住**：
- ✅ 攻击者拥有 IdP 的合法账号（哪怕是同组织内的一个低权限普通员工账号）
- ✅ IdP 本身存在漏洞（如会话固定、弱密码、未启用 MFA、开放注册等）
- ✅ IdP 配置了错误的 scope 导致返回了过多用户信息
- ✅ 攻击者先让**受害者**在 IdP 完成登录授权（再结合后续防线分析能否劫持到受害者的 code）

### 6.3 防线二：PKCE 校验

**实现有两条分支**（[sso_client.rs](src/sso_client.rs#L216-L225)）：

```
sso_pkce=true（默认）:  verifier 直接发给 IdP Token 端点，由 IdP 校验
sso_pkce=false:         Vaultwarden 本地计算 SHA256(verifier) == sso_auth.client_challenge
```

**能拦住**：
- ❌ 纯授权码劫持攻击：攻击者通过某种渠道（恶意 App、网络中间人、恶意浏览器扩展）截获了 IdP 返回的 `code`，但不知道对应的 `code_verifier` → `/connect/token` 兑换时 PKCE 校验失败
- ❌ 攻击者在 `/connect/authorize` 中填入自己的 `code_challenge`，但在 `/connect/token` 时不知道对应 `verifier` 的情况

**拦不住**：
- ✅ 攻击者完整走完自己的授权流程：自己发起 `/connect/authorize`（填入自己的 challenge）→ 自己在 IdP 登录 → 自己收到 code → 用自己知道的 verifier 兑换 token。此时 PKCE 是"自己校验自己"，不起防护作用
- ✅ 攻击者能同时控制授权发起和 code 兑换两个阶段（例如同一恶意客户端）
- ✅ `sso_pkce=false` 且攻击者能读数据库：直接从 `sso_auth.client_challenge` 字段反推是不可能的（SHA256 不可逆），但如果攻击者能写入数据库另当别论

### 6.4 防线三：浏览器绑定 Cookie（`VW_SSO_BINDING`）

**校验点**：仅在 `/connect/oidc-signin`（IdP 回调 Vaultwarden 时）执行一次（[identity.rs](src/api/identity.rs#L1214-L1223)）。

Cookie 属性（[identity.rs](src/api/identity.rs#L1294-L1302)）：
- `Path=/identity/connect/`（仅在该路径及其子路径下发送）
- `SameSite=Lax`（跨站 POST 不发送，但跨站 GET 顶级导航会发送）
- `HttpOnly`（JS 无法通过 `document.cookie` **读取**明文值，但 XSS 仍可在受害者浏览器上下文中通过 fetch/XHR 发起请求让浏览器自动带上该 Cookie——因此 HttpOnly 只能防止"Cookie 值被窃取后在攻击者环境中复用"，不能阻止 XSS 上下文中的即时攻击）
- `Secure`（仅 HTTPS 下发送，取决于当前请求是否 https）

校验逻辑：
```
Cookie 中的明文 binding_token → SHA-256 → 必须等于 DB 中 sso_auth.binding_hash
```

**能拦住**：
- ❌ **纯 CSRF 攻击**：第三方网站诱导受害者浏览器访问 `GET /connect/oidc-signin?code=X&state=Y`（注意这是 GET）。
  - 事实校准：`/connect/oidc-signin` 本身是 GET 端点，而 OAuth2 的 `state` 参数本身已经是标准 CSRF 防护机制——攻击者如果不知道 `state` 值就无法让 Vaultwarden 在 DB 中定位到对应的 `sso_auth` 记录。
  - 额外的 SSO_BINDING Cookie 提供了第二道防线：即使攻击者通过某种方式泄露了 `state`，也还需要受害者浏览器携带正确的 SSO_BINDING Cookie 才能通过校验（而 Cookie 不会被第三方网站读取或伪造）。
  - SameSite=Lax 在跨站顶级 GET 导航时**会发送 Cookie**，但仅靠 CSRF 无法构造 `state`，因此该场景仍被联合防御拦住。
- ❌ **跨设备/跨浏览器会话劫持**：攻击者在自己的浏览器发起授权（得到自己的 SSO_BINDING Cookie 和 state），然后诱导受害者浏览器回调 `/connect/oidc-signin?code=...&state=攻击者的state`。
  - 受害者浏览器不会携带攻击者浏览器的 SSO_BINDING Cookie → 哈希不匹配 → 被拦截。
- ❌ **攻击者只拿到 code 但拿不到 Cookie**：例如通过 Referer 泄露、日志泄露等方式拿到 code，但无法读取受害者浏览器的 Cookie。

**拦不住**：
- ✅ **同一浏览器内的 XSS 攻击**：HttpOnly 只是阻止 JS 读取 Cookie 明文，但如果攻击者已经能在受害者浏览器中执行 JS（XSS），可以直接通过 `fetch('/identity/connect/oidc-signin?...')` 发起请求，浏览器会自动带上 SSO_BINDING Cookie。但攻击者仍然需要知道受害者会话对应的 `state` 值才能成功完成回调——这取决于 XSS 是否能访问到当前页面上下文中的 `state`。
- ✅ **恶意浏览器扩展 / 本地恶意软件**：不受浏览器同源策略和 HttpOnly 限制，可以直接读取任意 Cookie 并伪造请求。
- ✅ **攻击者自己完整走完流程**：在自己的浏览器中发起授权，自己携带 Cookie 完成回调 → 校验当然通过。
- ✅ **Path  bypass 理论可能**：Cookie 的 Path 限制为 `/identity/connect/`，虽然目前回调端点就在该路径下，但如果将来有其他 SSO 相关端点不在此路径下可能有风险（当前版本不存在）。
- ✅ **SameSite=Lax 的固有局限性**：在顶级跨站导航（如点击 `<a href="...">`）时 Cookie 仍会发送，结合 `state` 泄露才能构成攻击。

### 6.5 防线四：Id Token 校验

**事实校准**：Id Token 的 claims 校验分两层完成，并非全部在 `id_token.claims()` 调用中一次性完成，也有部分校验（如 `iss`）在 `sso.rs` 的 `decode_token_claims` 中针对的是 **access_token / refresh_token**，不是 Id Token——这点之前的表述不准确。

真实的校验路径如下：

#### 校验 1：Id Token 本身（`openidconnect` crate 负责）

触发点：[sso_client.rs](src/sso_client.rs#L243) 调用 `id_token.claims(&self.vw_id_token_verifier(), &oidc_nonce)`。`openidconnect = "4.0.1"` 内置校验以下 claims：

| 校验项 | 说明 | 依据来源 |
|--------|------|----------|
| **签名 (signature)** | 必须是 IdP 私钥签发，公钥从 Discovery 拉取的 JWKS 中匹配（按 kid） | `openidconnect` crate 内置，由 `CoreProviderMetadata::discover_async` 填充密钥 |
| **issuer (iss)** | 必须与 Discovery 时使用的 `issuer_url` 精确一致 | `openidconnect` crate 内置（`CoreIdTokenVerifier` 默认行为） |
| **audience (aud)** | 必须包含 `sso_client_id`；若配置了 `SSO_AUDIENCE_TRUSTED` 正则，额外信任匹配该正则的任意值 | 默认校验：`openidconnect` 内置；扩展正则：[sso_client.rs](src/sso_client.rs#L273-L286) 的 `vw_id_token_verifier()` 通过 `set_other_audience_verifier_fn` 注入 |
| **nonce** | 必须与授权发起时生成、保存在 `sso_auth.nonce` 的随机值一致 | [sso_client.rs](src/sso_client.rs#L230) + `openidconnect` 内置 |
| **过期时间 (exp)** | JWT 标准校验，带 60 秒默认 leeway | `openidconnect` crate 内置（`jsonwebtoken` 默认行为） |
| **生效时间 (nbf / iat)** | 若存在则校验，否则忽略 | `openidconnect` crate 内置 |
| **authorized_party (azp)** | 若存在则校验与 client_id 一致 | `openidconnect` crate 内置 |

此外 Vaultwarden 还在 [sso_client.rs](src/sso_client.rs#L232-L234) 做了一层前置检查：**token_response 中必须存在 `id_token` 字段**，否则直接报错。

#### 校验 2：access_token / refresh_token（Vaultwarden 手动校验，注意不是 Id Token）

触发点：`sso::redeem()` 和 `sso::create_auth_tokens()` 调用 [sso::decode_token_claims()](src/sso.rs#L151-L171)，用于从 IdP 返回的 access_token / refresh_token 中提取 `nbf` / `exp` 以决定 Vaultwarden 自身 Token 的生命周期。该校验**不用于安全准入**，仅用于时间计算：

```rust
// sso.rs#L151-L171
// 仅校验 exp（过期）和 iss（issuer），不校验签名、audience、nonce 等
if validate_claim.exp < now - 60 { ... }          // exp 校验（60 秒 leeway）
if validate_claim.iss.ne(&CONFIG.sso_authority()) { ... }  // iss 校验（与 sso_authority 精确相等）
```

这部分使用 `jsonwebtoken::dangerous::insecure_decode`，明确跳过签名校验，仅做时间和 issuer 的粗粒度检查——因为真正的签名、audience 等安全校验已经由 openidconnect 在 Id Token 阶段完成。

#### 防御边界

**能拦住**：
- ❌ 攻击者自己伪造的 Id Token（没有 IdP 私钥，签名校验不通过）
- ❌ 其他 IdP 签发的合法 Token（issuer 与 Discovery 时记录的不一致）
- ❌ 过期的 Id Token（exp 校验不通过）
- ❌ audience 既不包含 `sso_client_id`、也不匹配 `SSO_AUDIENCE_TRUSTED` 正则的 Token
- ❌ nonce 不匹配的 Token（防止把为另一个会话签发的 Id Token 重放到当前会话）
- ❌ Token 响应中根本没有 `id_token` 字段的情况

**拦不住**：
- ✅ IdP 为攻击者本人签发的**合法** Id Token（iss/aud/nonce/签名/exp 全部合规）
- ✅ IdP 签发的 audience 虽然不是 `sso_client_id`，但恰好匹配 `SSO_AUDIENCE_TRUSTED` 正则的 Token（如果正则写得过宽，例如 `.*`，则等同于信任任意 audience）
- ✅ IdP 自身存在漏洞（如支持 `alg:none`、签名密钥泄露、Token 注入等）签发的任意 Token（属于 IdP 侧责任，不是 Vaultwarden 防线范围）

### 6.6 防线五：账号映射

**执行位置**：`sso_login()` 中 `SsoUser::find_by_identifier` → `SsoUser::find_by_mail` 的两级查找（[identity.rs](src/api/identity.rs#L208-L264)），再加上后续 redeem 时的关联写入。

**能拦住**：
- ❌ **Identifier 精确匹配不命中 + Email 也不命中 + 邮箱域名不在白名单**：攻击者 IdP 账号的邮箱在 Vaultwarden 中完全不存在，且未被 `signups_domains_whitelist` 放行 → 登录失败
- ❌ **想冒用他人邮箱但 IdP 返回的邮箱不匹配**：攻击者的 IdP 账号只能返回自己的邮箱，无法通过映射跳到其他用户
- ❌ **已有 SSO 用户同邮箱**：攻击者的 IdP identifier 是新的，但邮箱已被另一个 SSO identifier 占用 → 报错"Existing SSO user with same email"，防止两个 OIDC 账号共享一个 Vaultwarden 账号
- ❌ **`sso_signups_match_email=false`**：即使邮箱匹配，也不允许关联已有非 SSO 用户
- ❌ **IdP 不返回 `email_verified=true`**：默认情况下 `sso_allow_unknown_email_verification=false`，邮箱验证状态不明或明确为 false 都拒绝关联

**拦不住**：
- ✅ **Identifier 精确命中**（正常合法用户场景）：攻击者此前已用自己的 IdP 账号注册过，直接登录自己的账号
- ✅ **Email 匹配 + 邮箱已验证 + 允许关联**：攻击者的 IdP 账号邮箱恰好就是目标 Vaultwarden 用户的邮箱，且 IdP 返回 `email_verified=true` → 成功关联到目标用户账号并登录
  - 这在 IdP 与 Vaultwarden 使用同一套企业邮箱体系时通常是期望行为，但如果 IdP 允许用户任意自定义邮箱（且未强制验证）就会构成账号接管风险
- ✅ **两级都不命中 + 邮箱域名在白名单（或白名单为空）+ email_verified 合规**：自动创建新 Vaultwarden 账号
  - **事实校准**：SSO 自动注册的开关与普通注册的 `signups_allowed` **完全独立**。普通注册走 [identity.rs](src/api/identity.rs#L1050) 的 `CONFIG.is_signup_allowed()`（同时检查 `signups_allowed` 和 `signups_domains_whitelist`），而 SSO 自动注册走 [identity.rs](src/api/identity.rs#L270) 的 `CONFIG.is_email_domain_allowed()`（**只检查域名白名单，不检查 `signups_allowed`**）。
  - 具体规则（[config.rs](src/config.rs#L1495-L1516)）：
    - `signups_domains_whitelist` 非空 → 仅允许白名单内的域名注册（SSO 和普通注册都遵循）
    - `signups_domains_whitelist` 为空 → **所有邮箱域名都可以通过 SSO 自动注册**（因为 `is_email_domain_allowed` 在白名单为空时恒返回 true），但普通注册仍受 `signups_allowed` 控制
  - 因此，`signups_allowed=false` 只会禁用邮箱+密码的普通注册，**不会阻止 SSO 用户自动创建新账号**。要完全禁止 SSO 新用户注册，必须配合设置 `signups_domains_whitelist` 来限制允许的邮箱域名。
- ✅ **目标用户是被邀请的 Stub 用户**（`private_key.is_none()`）：邮箱匹配后会直接补齐 verified_at 和 name 并继续

### 6.7 综合攻击路径评估

基于以上分析，对常见攻击场景逐一判定：

| 攻击场景 | 是否可行 | 在哪一道防线被拦截 |
|----------|----------|-------------------|
| 攻击者完全没有 IdP 账号，想直接登录 Vaultwarden | ❌ 不可行 | 防线一（IdP 登录）：没有凭据拿不到 code |
| 攻击者有自己的合法 IdP 账号，想登录**自己的** Vaultwarden 账号 | ✅ 可行 | 全部防线正常放行（预期行为） |
| 攻击者有自己的合法 IdP 账号，想登录**他人的** Vaultwarden 账号（邮箱不同） | ❌ 不可行 | 防线五（账号映射）：identifier 和 email 都不匹配目标用户 |
| 攻击者有自己的合法 IdP 账号，IdP 返回的邮箱恰好等于目标用户邮箱且 email_verified=true | ✅ 可行（风险） | 防线五可能放行（取决于 `sso_signups_match_email` 配置）；这是 IdP 身份与 Vaultwarden 账号的信任边界 |
| 管理员设置了 `signups_allowed=false`，攻击者仍通过 SSO 自动创建新账号 | ✅ 可行（注意） | **不会被拦截**：SSO 自动注册走 `is_email_domain_allowed()`，不检查 `signups_allowed`；需配合 `signups_domains_whitelist` 才能限制 SSO 新用户 |
| 攻击者通过某种渠道截获了受害者的 `code`，但没有其他上下文 | ❌ 不可行 | 防线三（需要 SSO_BINDING Cookie 才能写入 DB 的 code_response）+ 防线二（需要 verifier 才能兑换 token） |
| 攻击者在受害者浏览器中发起 SSO 授权（XSS/CSRF），想获取受害者的 code | ❌ 极难 | 防线三 + OAuth2 `state` 参数本身的 CSRF 防护：攻击者需要同时知道 `state` 值，并让受害者浏览器携带正确的 SSO_BINDING Cookie；`SameSite=Lax` 对跨站 POST 提供额外保护 |
| 攻击者伪造 Id Token | ❌ 不可行 | 防线四（openidconnect crate 的签名校验不通过） |
| 攻击者用另一个 IdP 签发的合法 Token | ❌ 不可行 | 防线四（issuer 校验不通过，与 Discovery 时记录的 issuer_url 不匹配） |
| 攻击者跳过 `/sso/prevalidate`，不传 `ssoToken` | ✅ 可行（无影响） | 该参数本来就不校验，不影响任何后续防线 |

### 6.8 总结：各防线的职责分工

```
                    ┌──────────────────────────────────────────────────┐
                    │  sso_token / authorize 参数不校验的影响区域       │
                    │  仅限于"授权发起前"，不影响以下任何防线            │
                    └──────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────┐  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  防线一     │  │  防线二     │  │  防线三       │  │  防线四       │  │  防线五       │
│  IdP 登录   │─▶│  PKCE       │─▶│  浏览器绑定   │─▶│  Id Token 校验│─▶│  账号映射     │
│             │  │             │  │  Cookie      │  │              │  │              │
│  拦：无凭据 │  │  拦：code   │  │  拦：跨设备   │  │  拦：伪造/    │  │  拦：冒用     │
│  人/冒用   │  │  劫持       │  │  /CSRF 回调  │  │  篡改/过期   │  │  他人账号     │
│  IdP 凭据  │  │             │  │              │  │              │  │              │
└─────────────┘  └─────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
       │                │                  │                 │                  │
       ▼                ▼                  ▼                 ▼                  ▼
  需要攻破 IdP    需要同时拿到       需要读/写受害者       需要攻破 IdP       需要 IdP 返回
  本身或拿到      code 和 verifier   浏览器 Cookie         签名私钥或        受害者邮箱且
  受害者凭据                                             配置不当           验证通过
```

**核心结论**：`sso_token` 和 authorize 阶段多数参数的不校验确实让攻击者可以"少走一步路"（跳过 prevalidate、随便传参数），但**并不直接通向账号接管**。真正决定能否登录他人账号的是最后三道防线的组合——浏览器绑定 Cookie 保证了回调的浏览器上下文一致性，Id Token 校验保证了身份来源可信，账号映射决定了身份到 Vaultwarden 用户的归属关系。

### 四个需要特别关注的事实校准点

1. **Id Token 校验分层**：签名/issuer/audience/nonce/exp 由 `openidconnect = "4.0.1"` crate 在 `id_token.claims(&verifier, &nonce)` 调用时统一校验；`sso.rs` 中的 `decode_token_claims` 仅对 **access_token/refresh_token** 做粗粒度的 exp + iss 校验（用 `dangerous::insecure_decode` 跳过签名），目的是提取 Token 生命周期，并非 Id Token 的安全校验。

2. **HttpOnly 的真实边界**：只能阻止 JS 读取 Cookie 明文值，**无法阻止 XSS 上下文中的即时请求**——攻击者若已获得 XSS，仍可通过 `fetch()` 发起请求让浏览器自动带上 SSO_BINDING Cookie，只是需要同时知道当前会话的 `state` 值才能完成回调。

3. **CSRF 防护的真实机制**：`/connect/oidc-signin` 的 CSRF 防护主要依赖 OAuth2 标准的 `state` 参数（未知 state 无法定位 DB 中 sso_auth 记录），SSO_BINDING Cookie 是在此之上额外增加的浏览器上下文绑定层，并非 Vaultwarden 专门实现了"CSRF Token 机制"。

4. **SSO 自动注册独立于 `signups_allowed`**：SSO 新用户创建走 `CONFIG.is_email_domain_allowed()`（只检查域名白名单），普通注册走 `CONFIG.is_signup_allowed()`（同时检查 `signups_allowed` 和白名单）。因此：
   - `signups_allowed=false` 只禁用普通邮箱+密码注册，**不会阻止 SSO 用户自动创建账号**
   - 白名单为空时，**任何邮箱域名都能通过 SSO 自动注册**
   - 要完全禁止 SSO 新用户注册，必须通过 `signups_domains_whitelist` 明确限制允许的域名

剩余风险集中在 IdP 侧与配置侧：如果 IdP 允许攻击者控制返回的 email 字段且标记为已验证，或者 `SSO_AUDIENCE_TRUSTED` 正则配置过宽（如 `.*`），或者未设置域名白名单导致 SSO 注册完全开放，才可能造成实质安全问题。
