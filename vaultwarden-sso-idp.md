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

### 6.4 防线二与防线三的联合作用：state、PKCE、浏览器绑定 Cookie 与同源限制

本节把 state 流转、PKCE verifier、SSO_BINDING Cookie 和浏览器同源限制放在一起分析——它们在代码中是分层协作的，单独看任何一个都容易得出错误结论。

#### state 的完整代码事实（按执行顺序）

1. **state 是 authorize 入参，不由 Vaultwarden 生成**
   - 定义在 [identity.rs](src/api/identity.rs#L1247-L1268) 的 `AuthorizeData.state: OIDCState`，是客户端（Web/Desktop/Mobile/CLI）生成并通过 query 参数传入的，只是 `String` 的 newtype 包装。
   - Vaultwarden 对 state **不做任何校验**：不检查长度、格式、随机性、是否唯一冲突（冲突时 DB 的 `on_conflict` 会直接覆盖）。

2. **SsoAuth 以 state 为主键保存上下文**
   - 在 [sso_auth.rs](src/db/models/sso_auth.rs#L42-L57) 的 `SsoAuth` 结构体中，`state` 是 `#[diesel(primary_key(state))]`。
   - 保存时同时写入：`client_challenge`（PKCE code_challenge，也是客户端入参）、`nonce`（`sso::authorize_url` 内部随机生成）、`redirect_uri`（客户端入参/硬编码）、`binding_hash`（authorize 函数内随机生成的浏览器绑定哈希）。见 [identity.rs](src/api/identity.rs#L1286-L1292)。

3. **oidc-signin 用 state 回查 SsoAuth**
   - IdP 回调时把 state（经 Base64 编码）和 code 一起传回，Vaultwarden 在 [identity.rs](src/api/identity.rs#L1208-L1212) 解码 state 后调用 `SsoAuth::find(&state, conn)`：
     - 找不到 → 直接报错 `"Cannot retrieve sso_auth"`
     - 找到 → 进入 SSO_BINDING Cookie 校验

4. **Cookie 校验通过后写入 code_response**
   - [identity.rs](src/api/identity.rs#L1214-L1227)：SSO_BINDING 哈希匹配后，把 IdP 返回的 code 写入 `sso_auth.code_response`，然后**把 code 和 state 再次通过重定向传回给客户端**。

5. **/connect/token 阶段客户端再传 code 和 code_verifier**
   - 客户端拿到 code 后发起 `POST /identity/connect/token`（grant_type=authorization_code），传入 `code` 和 `code_verifier`（[identity.rs](src/api/identity.rs#L1091-L1147) 的 `ConnectData`）。
   - Vaultwarden 用 `SsoAuth::find_by_code(code)` 再次定位 sso_auth（[sso_auth.rs](src/db/models/sso_auth.rs#L122-L131)），然后在 PKCE 校验 + IdP Token 交换中使用 `code_verifier`。

#### 各防护机制的协作关系

```
客户端                          Vaultwarden                          IdP
  │                                │                                  │
  │ 1. GET /connect/authorize      │                                  │
  │    ?state=S                    │                                  │
  │    &code_challenge=C           │                                  │
  │ ─────────────────────────────▶ │                                  │
  │                                │ 2. 生成 SSO_BINDING Cookie B      │
  │                                │    生成 nonce N                   │
  │                                │    写入 SsoAuth(state=S,          │
  │                                │           challenge=C, nonce=N,   │
  │                                │           binding_hash=SHA256(B)) │
  │ 3. 302 → IdP authorize URL     │                                  │
  │    (携带 state=S, nonce=N,     │                                  │
  │     code_challenge=C)          │ ───────────────────────────────▶ │
  │                                │                                  │ 4. 用户在 IdP 登录
  │                                │                                  │ 5. IdP 校验 code_challenge
  │                                │                                  │    生成授权 code X
  │ 6. 302 → /connect/oidc-signin  │                                  │
  │    ?code=X&state=Base64(S)     │ ◀─────────────────────────────── │
  │ ─────────────────────────────▶ │                                  │
  │                                │ 7. SsoAuth::find(state=S)         │
  │                                │    校验 SHA256(Cookie B)          │
  │                                │       == sso_auth.binding_hash   │
  │                                │    写入 sso_auth.code_response=X  │
  │ 8. 302 → redirect_uri          │                                  │
  │    ?code=X&state=S             │                                  │
  │ ◀───────────────────────────── │                                  │
  │                                │                                  │
  │ 9. POST /connect/token         │                                  │
  │    grant_type=authorization_   │                                  │
  │    code                         │                                  │
  │    code=X & code_verifier=V    │                                  │
  │ ─────────────────────────────▶ │                                  │
  │                                │ 10. SsoAuth::find_by_code(X)      │
  │                                │     PKCE 校验:                    │
  │                                │     sso_pkce=false:               │
  │                                │       SHA256(V) == challenge C   │
  │                                │     sso_pkce=true:                │
  │                                │       把 V 发给 IdP 校验          │
  │                                │ 11. 向 IdP 兑换 Token             │
  │                                │ ───────────────────────────────▶ │
  │                                │                                  │ 12. IdP 校验 code X + verifier V
  │                                │ ◀─────────────────────────────── │
  │                                │     返回 id_token/access_token    │
  │                                │ 13. 校验 id_token(nonce=N, ...)   │
  │                                │ 14. 账号映射 → 返回 Vaultwarden   │
  │ ◀───────────────────────────── │     access_token / refresh_token  │
```

#### 逐攻击场景分析：攻击者能走到哪里

| 攻击场景 | 能走到第几步 | 被哪道防线拦截 | 代码依据 |
|----------|-------------|---------------|----------|
| **场景 A：纯跨站 CSRF**（第三方网站 evil.com 诱导受害者点击 `<a href="/identity/connect/oidc-signin?code=X&state=Y">`） | 到第 7 步（SsoAuth::find）之前 | state 不可知：evil.com 无法读取受害者浏览器上下文中的 state 值（state 由客户端生成，存在于 Bitwarden 客户端内存/URL 参数中，跨站不可读）。即使盲目猜 state，也无法通过后续 SSO_BINDING 哈希校验。 | `SsoAuth::find` 找不到或 Cookie 哈希不匹配 → [identity.rs](src/api/identity.rs#L1210-L1221) |
| **场景 B：跨站 CSRF + state 泄露**（攻击者通过 Referer、日志、URL 分享等渠道拿到了受害者的 state 值） | 到第 7 步的 SSO_BINDING 校验 | SSO_BINDING Cookie：SameSite=Lax 在跨站顶级 GET 导航时会发送 Cookie，state 也已泄露；但攻击者构造的 URL 中的 state=Y 对应的是 **攻击者自己的** sso_auth.binding_hash（攻击者的浏览器生成的），而受害者浏览器携带的是 **受害者自己的** SSO_BINDING Cookie → 哈希不匹配。 | SHA256 比对失败 → [identity.rs](src/api/identity.rs#L1218-L1221) |
| **场景 C：XSS（攻击者在 Vaultwarden 域名下有 JS 执行权限）** | 能走到第 8 步（oidc-signin 重定向完成），第 9-14 步取决于能否拿到 code_verifier | 浏览器绑定 Cookie：XSS 中通过 `fetch()` 发起请求，浏览器自动带上 SSO_BINDING Cookie；如果 XSS 能访问当前页面上下文中的 state 值，则可成功完成 oidc-signin 回调。但**到第 9 步还需要 `code_verifier`**——verifier 由 Bitwarden 客户端生成并保存在客户端内存中，如果 XSS 在 Web Vault 页面上下文中，可能通过 JS 读取内存中的 verifier；如果是在其他页面上下文中的 XSS，则拿不到 verifier，在第 10 步 PKCE 校验被拦截。 | PKCE 校验需要 verifier → [sso_client.rs](src/sso_client.rs#L216-L225) |
| **场景 D：中间人/网络窃听者**（能看到所有 HTTP 流量，假设 HTTPS 已被攻破或降级） | 能看到第 1-9 步的所有明文，但仍被第 10 步 PKCE 和第 13 步 Id Token 校验拦住 | 可以截获 state（第 1 步）、code（第 6 步），但：① **code_verifier** 在第 9 步从客户端发到 Vaultwarden，中间理论上能拿到，但拿到了也没用——IdP 的 code X 只能使用一次（OIDC 规范）；② **SSO_BINDING Cookie 的 Secure 属性**在 HTTPS 下即使网络层能看到 Cookie，攻击者在自己的浏览器中也无法伪造该 Cookie（因为 Secure）；③ 即使伪造了请求，Id Token 的签名校验仍然无法绕过。 | PKCE one-time code + Id Token 签名 |
| **场景 E：攻击者控制同一浏览器的恶意扩展/本地恶意软件** | 能走完全流程 | 浏览器扩展不受 SameSite/HttpOnly/同源限制，可以读取 state、code_verifier、Cookie，可以伪造任意请求。这种情况所有浏览器侧防护全部失效。 | 浏览器安全模型之外 |
| **场景 F：攻击者在自己的浏览器中完整走合法流程** | 能走完全流程（预期行为） | 所有防护机制都是针对"攻击者冒充受害者"设计的，对合法用户自己的登录流程不做拦截。 | 正常登录路径 |

#### 关键结论

- **state 本身不保证安全**：它只是一个数据库查找键，由客户端生成且不校验，单独不足以防御 CSRF；真正起作用的是 state + SSO_BINDING Cookie 的**组合**——state 确保回调能定位到正确的会话，Cookie 确保发起授权和接收回调的是同一浏览器。
- **PKCE 是 code 被盗后的最后一道防线**：即使 state 和 code 同时泄露（如 Referer 泄露），攻击者没有 code_verifier 也无法在 `/connect/token` 阶段兑换到 Token。
- **同源限制保护 /connect/token 端点**：如果攻击者试图从第三方网站用 AJAX 直接调用 `POST /identity/connect/token`，会被浏览器 CORS 策略拦截；但攻击者如果已经拿到 code 和 verifier，也可以在自己的非浏览器环境中直接发 POST，不受同源限制——这就是 PKCE 存在的意义。

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

| 攻击场景 | 是否可行 | 实际能走到哪一步 / 被哪道防线拦截 |
|----------|----------|----------------------------------|
| 攻击者完全没有 IdP 账号，想直接登录 Vaultwarden | ❌ 不可行 | 防线一（IdP 登录）：没有凭据拿不到 code |
| 攻击者有自己的合法 IdP 账号，想登录**自己的** Vaultwarden 账号 | ✅ 可行 | 全部防线正常放行（预期行为） |
| 攻击者有自己的合法 IdP 账号，想登录**他人的** Vaultwarden 账号（邮箱不同） | ❌ 不可行 | 防线五（账号映射）：identifier 和 email 都不匹配目标用户 |
| 攻击者有自己的合法 IdP 账号，IdP 返回的邮箱恰好等于目标用户邮箱且 email_verified=true | ✅ 可行（风险） | 防线五可能放行（取决于 `sso_signups_match_email` 配置）；这是 IdP 身份与 Vaultwarden 账号的信任边界 |
| 管理员设置了 `signups_allowed=false`，攻击者仍通过 SSO 自动创建新账号 | ✅ 可行（注意） | **不会被拦截**：SSO 自动注册走 `is_email_domain_allowed()`，不检查 `signups_allowed`；需配合 `signups_domains_whitelist` 才能限制 SSO 新用户 |
| **场景 A：纯跨站 CSRF**（第三方网站诱导受害者点击回调链接） | ❌ 不可行 | state 不可知：`SsoAuth::find` 找不到记录（跨站无法读取受害者客户端内存中的 state 值） |
| **场景 B：跨站 CSRF + state 泄露**（攻击者通过 Referer/日志拿到了受害者的 state） | ❌ 不可行 | SSO_BINDING Cookie 哈希不匹配：state 对应攻击者自己的 sso_auth.binding_hash，但受害者浏览器携带的是受害者自己的 Cookie |
| **场景 C：仅截获 code，没有其他上下文**（如 Referer 泄露 code） | ❌ 不可行 | 两层拦截：① oidc-signin 阶段需要 state + SSO_BINDING Cookie 才能把 code 写入 DB；② 即使跳过写入直接调 /connect/token，也需要 PKCE verifier 才能兑换 |
| **场景 D：XSS 在 Vaultwarden 域名下执行 JS**（能拿到 state 且 fetch 自动带 Cookie） | ⚠️ 部分可行 | 能走完 oidc-signin 回调（第 1-8 步），但第 9 步需要 PKCE code_verifier；verifier 在 Bitwarden 客户端内存中，若 XSS 不在 Web Vault SSO 流程页面上下文中则拿不到，PKCE 校验拦截 |
| **场景 E：中间人/网络窃听者**（HTTPS 假设已攻破，能看到所有流量） | ❌ 不可行 | ① IdP 的 code 只能用一次（OIDC 规范），中间人即使拿到 code+verifier 也会和合法客户端竞争；② SSO_BINDING Cookie 的 Secure 属性使攻击者无法在自己浏览器中伪造；③ Id Token 签名无法伪造 |
| **场景 F：恶意浏览器扩展/本地恶意软件**（不受浏览器安全模型限制） | ✅ 可行 | 能读取 Cookie、state、code_verifier，能伪造任意请求，所有浏览器侧防护全部失效 |
| 攻击者伪造 Id Token | ❌ 不可行 | 防线四（openidconnect crate 的签名校验不通过） |
| 攻击者用另一个 IdP 签发的合法 Token | ❌ 不可行 | 防线四（issuer 校验不通过，与 Discovery 时记录的 issuer_url 不匹配） |
| 攻击者跳过 `/sso/prevalidate`，不传 `ssoToken` | ✅ 可行（无影响） | 该参数本来就不校验，不影响任何后续防线 |

### 6.8 总结：各防线的职责分工

```
                    ┌──────────────────────────────────────────────────────┐
                    │    sso_token / authorize 参数不校验的影响区域         │
                    │    仅限于"授权发起前"，不影响以下任何防线              │
                    └──────────────────────────────────────────────────────┘
                                               │
                                               ▼
┌─────────────┐  ┌──────────────────────────────────────────────┐  ┌──────────────┐  ┌──────────────┐
│  防线一     │  │  防线二 + 防线三：state + PKCE + 浏览器绑定     │  │  防线四       │  │  防线五       │
│  IdP 登录   │─▶│  Cookie + 同源限制（联合作用，缺一不可）       │─▶│  Id Token 校验│─▶│  账号映射     │
│             │  │                                              │  │              │  │              │
│  拦：无凭据 │  │  state：只是 DB 查找键（客户端生成，不校验）   │  │  拦：伪造/    │  │  拦：冒用     │
│  人/冒用   │  │  SSO_BINDING Cookie：保证同一浏览器上下文       │  │  篡改/过期   │  │  他人账号     │
│  IdP 凭据  │  │  PKCE verifier：code 被盗后的最后一道防线       │  │              │  │              │
│             │  │  同源限制：阻止跨站 AJAX 调用 /connect/token   │  │              │  │              │
└─────────────┘  └──────────────────────────────────────────────┘  └──────────────┘  └──────────────┘
       │                              │                                       │                  │
       ▼                              ▼                                       ▼                  ▼
  需要攻破 IdP              需同时突破：state 可知 +                          需要攻破 IdP       需要 IdP 返回
  本身或拿到                Cookie 可伪造 + verifier 可获取                    签名私钥或        受害者邮箱且
  受害者凭据                                                                   配置不当           验证通过
```

**核心结论**：`sso_token` 和 authorize 阶段多数参数的不校验确实让攻击者可以"少走一步路"（跳过 prevalidate、随便传参数），但**并不直接通向账号接管**。state 本身只是数据库查找键，不单独承担安全防护职责；真正构成"攻击者无法冒充受害者"的，是 state + SSO_BINDING Cookie + PKCE verifier + 同源限制的**联合防线**——state 让回调能定位到正确会话，Cookie 保证发起授权和接收回调的是同一浏览器，PKCE 保证即使 code 泄露也无法兑换 Token，同源限制让跨站攻击者无法直接调用 Token 端点。最后 Id Token 校验保证身份来源可信，账号映射决定身份到 Vaultwarden 用户的归属关系。

### 五个需要特别关注的事实校准点

1. **Id Token 校验分层**：签名/issuer/audience/nonce/exp 由 `openidconnect = "4.0.1"` crate 在 `id_token.claims(&verifier, &nonce)` 调用时统一校验；`sso.rs` 中的 `decode_token_claims` 仅对 **access_token/refresh_token** 做粗粒度的 exp + iss 校验（用 `dangerous::insecure_decode` 跳过签名），目的是提取 Token 生命周期，并非 Id Token 的安全校验。

2. **HttpOnly 的真实边界**：只能阻止 JS 通过 `document.cookie` 读取 Cookie 明文值，**无法阻止 XSS 上下文中的即时请求**——攻击者若已获得 XSS，仍可通过 `fetch()` 发起请求让浏览器自动带上 SSO_BINDING Cookie，只是需要同时知道当前会话的 `state` 值才能完成回调；后续兑换 Token 还需要 PKCE code_verifier。

3. **state 与 CSRF 防护的真实机制**：
   - `state` 是**客户端生成并传入** authorize 的参数（[identity.rs](src/api/identity.rs#L1247-L1268)），Vaultwarden 对其不做任何校验（不检查长度、格式、随机性、唯一性冲突），只是把它当作 `SsoAuth` 的主键保存，回调时用它回查 DB。
   - 因此 `state` **本身不直接承担 CSRF 防护职责**，它只是让 Vaultwarden 能找到对应会话的"索引"。真正的防护来自 state + SSO_BINDING Cookie 的组合：即使攻击者通过某种渠道知道了 state 值，也无法让受害者浏览器携带正确的 SSO_BINDING Cookie（攻击者的 state 对应攻击者自己的 binding_hash，与受害者浏览器的 Cookie 哈希不匹配）。
   - `/connect/token` 端点的跨站 AJAX 调用则被浏览器**同源策略**（CORS）拦截，这是浏览器默认行为，Vaultwarden 没有额外实现"CSRF Token 机制"。

4. **PKCE 的关键作用**：PKCE verifier 由 Bitwarden 客户端在本地生成并保存在内存中，**从不经过网络传输到 Vaultwarden**（直到第 9 步 `/connect/token` 才通过 HTTPS POST 发送）。因此即使攻击者通过 Referer 泄露、URL 分享等方式拿到了 code 和 state，没有 verifier 也无法兑换到 Token——这是 code 被盗场景下的最后一道防线。

5. **SSO 自动注册独立于 `signups_allowed`**：SSO 新用户创建走 `CONFIG.is_email_domain_allowed()`（只检查域名白名单），普通注册走 `CONFIG.is_signup_allowed()`（同时检查 `signups_allowed` 和白名单）。因此：
   - `signups_allowed=false` 只禁用普通邮箱+密码注册，**不会阻止 SSO 用户自动创建账号**
   - 白名单为空时，**任何邮箱域名都能通过 SSO 自动注册**
   - 要完全禁止 SSO 新用户注册，必须通过 `signups_domains_whitelist` 明确限制允许的域名

剩余风险集中在 IdP 侧与配置侧：如果 IdP 允许攻击者控制返回的 email 字段且标记为已验证，或者 `SSO_AUDIENCE_TRUSTED` 正则配置过宽（如 `.*`），或者未设置域名白名单导致 SSO 注册完全开放，才可能造成实质安全问题。
