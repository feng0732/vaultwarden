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
