# Vaultwarden 限流与安全防护协作机制

## 概述

Vaultwarden 通过多层次的安全防护机制实现完整的安全体系，包括：
- **请求限流**：基于 IP 的速率限制，防止暴力破解
- **登录限制**：多种登录方式的统一限流保护（含刷新令牌的例外设计和旧格式兼容）
- **安全 Headers**：CSP、CORS 等 Web 安全标准，含精细的例外规则（WebSocket 特殊处理）
- **代理支持**：正确处理反向代理场景下的真实 IP，含信任边界分析
- **认证守卫分层**：从 JWT 验证到安全戳校验的多级防线

---

## 一、限流核心实现

### 1.1 限流模块结构（ratelimit.rs）

使用 `governor` crate 实现令牌桶算法的限流：

```rust
type Limiter<T = IpAddr> = RateLimiter<T, DashMapStateStore<T>, DefaultClock>;
```

**两个独立的限流器：**

| 限流器 | 配置项 | 默认值 | 说明 |
|--------|--------|--------|------|
| **登录限流** | `login_ratelimit_seconds` | 60秒 | 登录请求平均速率 |
| | `login_ratelimit_max_burst` | 10 | 允许的突发请求数 |
| **管理后台限流** | `admin_ratelimit_seconds` | 300秒 | 管理后台请求平均速率 |
| | `admin_ratelimit_max_burst` | 3 | 允许的突发请求数 |

**关键实现细节：** 两个限流器均使用 `LazyLock` 延迟初始化，全局共享同一个 `DashMap` 状态存储。这意味着同一 IP 的所有登录类型请求（密码、SSO、API Key、2FA 邮件发送）共享同一个令牌桶，而管理后台使用完全独立的桶。

### 1.2 限流检查函数

```rust
pub fn check_limit_login(ip: &IpAddr) -> Result<(), Error>
// 触发限流 → err_code!("Too many login requests", 429)

pub fn check_limit_admin(ip: &IpAddr) -> Result<(), Error>
// 触发限流 → err_code!("Too many admin requests", 429)
```

---

## 二、限流覆盖范围——完整调用点分析

对整个 API 目录搜索 `ratelimit::check_limit`，确认限流的**实际覆盖范围**：

### 2.1 被限流保护的入口（5 处）

| # | 路由函数 | 限流器 | 文件位置 |
|---|---------|--------|---------|
| 1 | `password_login` | `check_limit_login` | api/identity.rs |
| 2 | `sso_login` | `check_limit_login` | api/identity.rs |
| 3 | `api_key_login` | `check_limit_login` | api/identity.rs |
| 4 | `send_email_login` | `check_limit_login` | api/core/two_factor/email.rs |
| 5 | `post_admin_login` | `check_limit_admin` | api/admin.rs |

### 2.2 未被限流保护的关键入口——refresh_login

`refresh_login` 是 `grant_type="refresh_token"` 的处理函数。观察入口分发逻辑：

```rust
// api/identity.rs
"refresh_token" => {
    check_is_some(data.refresh_token.as_ref(), "refresh_token cannot be blank")?;
    refresh_login(data, &conn, &client_header.ip).await  // ← 无 check_limit_login
}
```

**refresh_login 不受限流保护的设计原因：**

1. **已有强认证保护**：`refresh_login` 调用 `auth::refresh_tokens`，该函数必须：
   - 提供有效的 refresh_token
   - 通过 JWT 验证或旧格式 token 回退逻辑（见下文）
   - 在数据库中查找匹配的 Device 记录
   - 关联到有效 User

2. **令牌本身即限流器**：refresh_token 是一个有有效期的凭据（默认 30 天，移动端 90 天），且绑定到特定设备。攻击者需要先窃取有效的 refresh_token 才能发起请求，无法像密码登录那样随意尝试。

3. **避免影响正常用户体验**：已认证用户的 token 刷新是高频操作（access_token 默认 2 小时过期），限流会严重影响使用流畅度。

4. **失败成本高**：无效的 refresh_token 会在验证阶段就被拒绝，不会触发任何数据库查询，对服务器压力极小。

### 2.3 旧格式 token 兼容——refresh_tokens 的回退逻辑

`refresh_tokens` 函数中有一个关键的**向后兼容**设计：

```rust
// auth.rs
let refresh_claims = match decode_refresh(refresh_token) {
    Err(err) => {
        error!("Failed to decode {} refresh_token: {refresh_token}: {err:?}", ip.ip);
        RefreshJwtClaims {
            nbf: 0,
            exp: 0,
            iss: String::new(),
            sub: AuthMethod::Password,   // ← 硬编码为 Password
            device_token: refresh_token.into(),  // ← 直接使用原始 token
            token: None,
        }
    }
    Ok(claims) => claims,
};

let Some(mut device) = Device::find_by_refresh_token(&refresh_claims.device_token, conn).await else {
    err!("Invalid refresh token")
};
```

**`decode_refresh` 的验证逻辑**（auth.rs:106-127）：

```rust
pub fn decode_jwt<T: DeserializeOwned>(token: &str, issuer: String) -> Result<T, Error> {
    let mut validation = jsonwebtoken::Validation::new(JWT_ALGORITHM);
    validation.leeway = 30;
    validation.validate_exp = true;   // ← 验证过期
    validation.validate_nbf = true;   // ← 验证生效时间
    validation.set_issuer(&[issuer]); // ← 验证签发者
    // RSA 签名验证内置在 jsonwebtoken::decode 中
}
```

**`Device::find_by_refresh_token` 的查询逻辑**（db/models/device.rs:222-224）：

```rust
pub async fn find_by_refresh_token(refresh_token: &str, conn: &DbConn) -> Option<Self> {
    conn.run(move |conn| devices::table
        .filter(devices::refresh_token.eq(refresh_token))
        .first::<Self>(conn).ok())
    .await
}
```

**关键发现：数据库查询的参数来源不同！**

| 格式 | `refresh_claims.device_token` 的值 | 数据库匹配的字段 |
|------|-------------------------------------|----------------|
| **新 JWT** | JWT claims 中的 `device_token`（可能是加密/哈希后的值） | `devices.refresh_token` |
| **旧 Base64** | 原始 token 字符串本身（`refresh_token.into()`） | `devices.refresh_token` |

- 新格式：JWT 解码成功 → 取 claims 中携带的 `device_token` → 用它匹配数据库
- 旧格式：JWT 解码失败 → 直接用原始 token 字符串匹配数据库
- 无论哪种格式，最终都是 `WHERE refresh_token = ?` 查询数据库，这是**共同的底线防线**

**AuthMethod 分支的影响**：

旧格式 token 的 `sub` 硬编码为 `AuthMethod::Password`，这会在后续 AuthMethod 匹配中产生安全决策：

```rust
let auth_tokens = match refresh_claims.sub {
    AuthMethod::Sso if CONFIG.sso_enabled() && CONFIG.sso_auth_only_not_session() => { ... }
    AuthMethod::Sso if CONFIG.sso_enabled() => { sso::exchange_refresh_token(...) }
    AuthMethod::Sso => err!("SSO is now disabled, Login again using email and master password"),
    AuthMethod::Password if CONFIG.sso_enabled() && CONFIG.sso_only() => err!("SSO is now required, Login again"),
    AuthMethod::Password => AuthTokens::new(&device, &user, refresh_claims.sub, client_id),
    _ => err!("Invalid auth method, cannot refresh token"),
};
```

**重要安全行为：**
- 旧格式 token（`sub=Password`）在 `sso_only=true` 时会被拒绝 → 管理员切换为 SSO Only 模式后旧 token 立即失效
- 旧格式 token 永远不会走 SSO 分支 → 即使数据库中的 Device 是通过 SSO 登录创建的，旧格式 token 也走 Password 分支
- 这意味着旧格式 token 在 SSO 环境下的行为可能与预期不符，但偏向**更安全**（SSO Only 会拒绝旧 token）

**安全边界分析：**

| 验证阶段 | 新 JWT 格式 | 旧 Base64 格式 | 说明 |
|----------|------------|---------------|------|
| RSA 签名 | ✓ 验证 | ✗ 跳过 | 旧格式直接构造 claims |
| 过期时间 | ✓ 验证 | ✗ 跳过 | `exp: 0` 不会被检查 |
| 生效时间 | ✓ 验证 | ✗ 跳过 | `nbf: 0` 不会被检查 |
| 签发者 | ✓ 验证 | ✗ 跳过 | `iss: ""` 不会被检查 |
| 数据库 Device 查询 | ✓ 通过 `device_token` | ✓ 通过原始 token | **共同底线** |
| 关联 User 查询 | ✓ 验证 | ✓ 验证 | **共同底线** |
| AuthMethod 匹配 | 使用 JWT 中的真实值 | 硬编码 `Password` | 旧格式可能被 `sso_only` 拒绝 |

**潜在风险：** 旧格式 token 没有过期时间检查，理论上永久有效。但实际上：
- 用户修改密码 → security_stamp 变更 → 旧 token 对应的 Device 刷新 → 旧 token 失效
- 管理员启用 `sso_only` → `sub=Password` 被拒绝 → 旧 token 失效
- 用户主动清除设备 → Device 记录删除 → 旧 token 失效

```
┌─────────────────────────────────────────────────────────────────────┐
│  refresh_tokens 验证流程                                             │
│                                                                       │
│  refresh_token 输入                                                   │
│       │                                                               │
│       ├─→ decode_refresh(JWT 解码)                                    │
│       │    ├─ 成功 → claims = JWT 签名验证过的 claims                  │
│       │    │   device_token = JWT 内嵌值                               │
│       │    │   sub = JWT 内嵌的 AuthMethod（Password 或 Sso）          │
│       │    │                                                           │
│       │    └─ 失败 → claims = 构造假 claims                            │
│       │        device_token = 原始 token 字符串本身 ★                  │
│       │        sub = AuthMethod::Password（硬编码）★                   │
│       │                                                               │
│       └─→ Device::find_by_refresh_token(claims.device_token)          │
│            │  新格式用 JWT 内嵌的 device_token 查数据库                 │
│            │  旧格式用原始 token 字符串直接查数据库                     │
│            │                                                           │
│            ├─ 没找到 → "Invalid refresh token"                        │
│            └─ 找到 → User 查询                                         │
│                 └─→ AuthMethod 分支匹配                                │
│                      sub=Password + sso_only → 拒绝                   │
│                      sub=Password → 生成 AuthTokens                    │
│                      sub=Sso + sso_enabled → SSO 交换                 │
│                      sub=Sso + !sso_enabled → 拒绝                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、安全 Headers 防护——例外规则详解

### 3.1 AppHeaders Fairing（util.rs）

`AppHeaders` 是一个 Rocket Response Fairing，在**每个响应**上设置安全头。其判断逻辑有三层例外：

### 3.2 例外场景 1：WebSocket 连接

```rust
// util.rs
if req_uri_path.ends_with("notifications/hub") || 
   req_uri_path.ends_with("notifications/anonymous-hub") {
    match (req_headers.get_one("connection"), req_headers.get_one("upgrade")) {
        (Some(c), Some(u))
            if c.to_lowercase().contains("upgrade") && u.to_lowercase().contains("websocket") =>
        {
            res.remove_header("X-Frame-Options");
            res.remove_header("X-Content-Type-Options");
            res.remove_header("Permissions-Policy");
            return;  // ← 直接返回，不设置任何其他安全 Header
        }
        (_, _) => (),  // 非 WebSocket 的 hub 请求正常设置 Header
    }
}
```

**注意细节（之前理解有偏差）：**
- 不是所有对 `/notifications/hub` 的请求都跳过安全头
- 只有同时携带 `Connection: Upgrade` **包含** upgrade 且 `Upgrade: websocket` **包含** websocket 的请求才跳过
- 代码用 `contains()` 而不是 `==` 精确匹配，兼容不同代理可能添加的额外值
- 跳过的 Header 只有三个：X-Frame-Options、X-Content-Type-Options、Permissions-Policy
- `return` 语句意味着 CSP、CORP、Referrer-Policy 等后续 Header **也全部不设置**
- 非 WebSocket 的 hub 请求（如普通 GET 探测）会正常设置安全 Header

### 3.3 例外场景 2：MFA connector.html

```rust
// util.rs
if req_uri_path.ends_with("connector.html") {
    res.remove_header("X-Frame-Options");  // 仅移除 X-Frame-Options
    // 不设置 CSP
} else {
    // 正常设置 CSP + X-Frame-Options
}
```

**原因**：WebAuthn 和 Duo 等 MFA 方案需要通过 iframe 弹窗完成验证，CSP 和 X-Frame-Options 会阻断这一流程。

### 3.4 例外场景 3：图片与 icon_external 路由

```rust
// util.rs
let mut is_image = true;
if !(res.headers().get_one("Content-Type").is_some_and(|v| v.starts_with("image/"))
    || req.route().is_some_and(|v| v.name.as_deref() == Some("icon_external")))
{
    is_image = false;
    res.set_raw_header("Cross-Origin-Resource-Policy", "same-origin");
}
```

两种情况不设置 `Cross-Origin-Resource-Policy`：
1. **Content-Type 为 image/** 的响应
2. **路由名为 `icon_external`** 的请求

**原因**：Bitwarden 桌面客户端需要跨域下载图标，设置 `CORP: same-origin` 会导致图标加载失败。图片类型使用独立的**严格 CSP**（禁止所有 script/frame/object）来弥补 CORP 缺失带来的风险。

### 3.5 安全 Header 例外规则汇总（修正版）

| 场景 | 触发条件 | 跳过的 Header | 原因 |
|------|---------|--------------|------|
| **WebSocket 握手** | URI 以 `/notifications/hub` 或 `/notifications/anonymous-hub` 结尾 **且**<br>Connection Header 包含 "upgrade" **且**<br>Upgrade Header 包含 "websocket" | X-Frame-Options<br>X-Content-Type-Options<br>Permissions-Policy<br>+ 后续所有未设置的 Header（CSP 等） | 反向代理/Cloudflare 对 WebSocket 协议的兼容性问题 |
| **connector.html** | URI 以 `connector.html` 结尾 | CSP 不设置<br>X-Frame-Options 移除 | MFA iframe 弹窗验证需要 |
| **图片响应** | Content-Type 以 `image/` 开头 | Cross-Origin-Resource-Policy | 桌面客户端图标下载，使用严格图片 CSP 补偿 |
| **icon_external 路由** | Rocket 路由名称为 `icon_external` | Cross-Origin-Resource-Policy | 桌面客户端外部图标下载，使用严格图片 CSP 补偿 |

### 3.6 管理后台诊断页自检

`admin_diagnostics.js` 中的 `checkSecurityHeaders` 函数会自动检测安全 Header 是否正确设置：
- **API 调用**：检查所有标准安全 Header
- **2FA Connector**：验证 `x-frame-options` 和 `content-security-policy` **不应出现**
- **HTTP 错误响应**：验证不被反向代理覆盖

---

## 四、CORS 跨域配置——精确实现（修正版）

### 4.1 Cors Fairing 的执行时机

Cors 是 **Response Fairing**（`Kind::Response`），在 Rocket 完成**路由匹配、请求守卫执行、业务逻辑处理**之后，构造响应时才执行。这意味着：

1. **Cors 不阻止请求进入路由处理函数**
2. **数据库查询、业务逻辑在 Cors 之前已经执行**
3. **Cors 只在响应上添加或修改 Header**
4. **浏览器根据这些 Header 决定是否将响应暴露给 JavaScript**

### 4.2 Origin 匹配逻辑

```rust
// util.rs
pub struct Cors();

impl Cors {
    fn get_allowed_origin(headers: &HeaderMap<'_>) -> Option<String> {
        let origin = Cors::get_header(headers, "Origin");
        let safari_extension_origin = "file://";
        let desktop_custom_file_origin = "bw-desktop-file://bundle";

        if origin == CONFIG.domain_origin()
            || origin == safari_extension_origin
            || origin == desktop_custom_file_origin
            || (CONFIG.sso_enabled() && origin == CONFIG.sso_authority())
        {
            Some(origin)  // ← 返回精确匹配的 Origin，而非 *
        } else {
            None
        }
    }
}
```

**白名单的四个来源：**

| 来源 | 值 | 条件 |
|------|----|------|
| `CONFIG.domain_origin()` | 配置的域名（如 `https://vault.example.com`） | 始终 |
| Safari 扩展 | `"file://"` | 始终 |
| 桌面客户端 | `"bw-desktop-file://bundle"` | 始终 |
| SSO Authority | 配置的 SSO 域名 | `CONFIG.sso_enabled()` 为 true |

**关键：`==` 是精确相等**，不是前缀匹配或包含匹配。Origin 必须**完全匹配**白名单中的某一项。

### 4.3 on_response 的精确行为

```rust
// util.rs
async fn on_response<'r>(&self, request: &'r Request<'_>, response: &mut Response<'r>) {
    let req_headers = request.headers();

    // 第一步：Origin 匹配时设置 Allow-Origin
    if let Some(origin) = Cors::get_allowed_origin(req_headers) {
        response.set_header(Header::new("Access-Control-Allow-Origin", origin));
    }
    // Origin 不匹配时：不设置 Allow-Origin Header
    // → 浏览器收到没有 Allow-Origin 的响应，拒绝将响应暴露给 JS

    // 第二步：处理 OPTIONS 预检请求
    if request.method() == Method::Options {
        let req_allow_headers = Cors::get_header(req_headers, "Access-Control-Request-Headers");
        let req_allow_method = Cors::get_header(req_headers, "Access-Control-Request-Method");

        response.set_header(Header::new("Access-Control-Allow-Methods", req_allow_method));
        response.set_header(Header::new("Access-Control-Allow-Headers", req_allow_headers));
        response.set_header(Header::new("Access-Control-Allow-Credentials", "true"));
        response.set_status(Status::Ok);
        response.set_header(ContentType::Plain);
        response.set_sized_body(Some(0), Cursor::new(""));  // ← 空响应体
    }
}
```

### 4.4 CORS 与数据库查询的关系

**Response Fairing 的执行时序：**

```
请求到达
  │
  ├─ 1. Rocket 路由匹配
  ├─ 2. FromRequest 守卫执行（ClientIp、Headers 等）
  ├─ 3. 路由处理函数执行 ← 数据库查询在这里发生
  │     （无论 Origin 是否匹配，业务逻辑都已执行）
  │
  ├─ 4. Response Fairing 执行 ← Cors 在这里
  │     ├─ Origin 匹配 → 设置 Allow-Origin
  │     └─ Origin 不匹配 → 不设置 Allow-Origin
  │
  └─ 5. 响应返回给客户端
       ├─ 浏览器检查 Allow-Origin
       │   ├─ 有 → 允许 JS 读取响应
       │   └─ 无 → 拒绝 JS 读取响应（但服务器已执行完毕）
       └─ 非浏览器客户端 → 完全忽略 CORS
```

**安全含义：**

- CORS 保护的是**浏览器端的数据读取**，不是服务器端的操作执行
- 恶意网站向 Vaultwarden 发起跨域请求时：
  - 如果 Origin 不匹配白名单 → 浏览器阻止 JS 读取响应 → 恶意网站拿不到数据
  - 但服务器**已经执行了**数据库查询等操作 → 这是 CSRF 的防护领域，不是 CORS 的职责
- CORS 与 CSRF Token 是互补的：CORS 防跨域数据泄露，CSRF Token 防跨域操作执行

### 4.5 OPTIONS 预检的特殊行为

**OPTIONS 请求的处理细节：**

- `Access-Control-Allow-Methods`：直接**回显** `Access-Control-Request-Method`
- `Access-Control-Allow-Headers`：直接**回显** `Access-Control-Request-Headers`
- 设置了 `Access-Control-Allow-Credentials: true` → 必须配合具体 Origin（不能是 `*`）
- 返回 200 OK + 空响应体

**回显设计的安全含义：** 任何请求方法和自定义 Header 都被允许（只要 Origin 匹配），这是一种"宽松许可"策略。安全性完全依赖 Origin 白名单的精确性——如果白名单中的一个 Origin 被攻陷，攻击者可以发起任意方法和 Header 的跨域请求。

**OPTIONS 请求是否经过路由处理？** 是的。Rocket 的路由处理在 Fairing 之前执行。但 Rocket 默认没有匹配 OPTIONS 方法的路由，所以 OPTIONS 请求通常在路由匹配阶段就得到 404 响应，然后 Cors Fairing 将其改写为 200 OK + CORS Header + 空响应体。

---

## 五、WebSocket 鉴权——精确代码路径（修正版）

### 5.1 两个 WebSocket 端点

| 端点 | 认证方式 | 用途 |
|------|---------|------|
| `/notifications/hub` | `decode_login` 验证 access_token（JWT） | 已登录用户的实时同步 |
| `/notifications/anonymous-hub` | URL token 直接用作订阅标识 | 登录请求推送（移动端扫码登录） |

### 5.2 已认证 WebSocket（hub）——完整鉴权链

```rust
// api/notifications.rs
#[get("/hub?<data..>")]
fn websockets_hub<'r>(
    ws: WebSocket,
    data: WsAccessToken,           // ← URL 查询参数解析
    ip: ClientIp,                  // ← FromRequest 守卫，提取 IP
    header_token: WsAccessTokenHeader,  // ← FromRequest 守卫，解析 Header
) -> Result<rocket_ws::Stream!['r>, Error> {
```

**步骤 1：WsAccessTokenHeader 守卫（永远成功）**

```rust
// auth.rs
pub struct WsAccessTokenHeader {
    pub access_token: Option<String>,
}

impl<'r> FromRequest<'r> for WsAccessTokenHeader {
    type Error = ();

    async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error> {
        let access_token = match request.headers().get_one("Authorization") {
            Some(a) => a.rsplit("Bearer ").next().map(String::from),
            None => None,
        };
        Outcome::Success(Self { access_token })  // ← 永远 Success，None 也算成功
    }
}
```

**关键：`WsAccessTokenHeader` 不是认证守卫！** 它永远返回 `Outcome::Success`，即使 `access_token` 为 `None`。与 `Headers` 守卫（JWT 验证失败时返回 `Outcome::Error`）完全不同。这是有意的设计：WebSocket 的 token 可能在 URL 参数中，Header 中没有 token 不应阻止请求进入路由处理。

**步骤 2：Token 提取优先级（路由处理函数内）**

```rust
let token = if let Some(token) = data.access_token {
    token            // 优先：URL 查询参数 ?access_token=...
} else if let Some(token) = header_token.access_token {
    token            // 次选：Authorization: Bearer ... Header
} else {
    err_code!("Invalid claim", 401)  // 两者都没有 → 401
};
```

**为什么 URL 参数优先？** 浏览器的 WebSocket API（`new WebSocket(url)`）不支持设置自定义 Header。浏览器客户端只能通过 URL 传递 token。

**步骤 3：JWT 验证（路由处理函数内）**

```rust
let Ok(claims) = crate::auth::decode_login(&token) else {
    err_code!("Invalid token", 401)
};
```

`decode_login` 调用 `decode_jwt`，执行完整的 JWT 验证链：
- RSA 签名验证（使用服务器公钥）
- 过期时间验证（`exp`，30 秒容差）
- 生效时间验证（`nbf`）
- 签发者验证（`iss` = `vaultwarden`）

**步骤 4：使用 claims.sub 注册连接**

```rust
let entry_uuid = uuid::Uuid::new_v4();
let (tx, rx) = tokio::sync::mpsc::channel::<Message>(100);
users.map.entry(claims.sub.to_string()).or_default().push((entry_uuid, tx));
```

`claims.sub` 的类型是 `UserId`（不是普通字符串），在 `LoginJwtClaims` 中定义：

```rust
// auth.rs
pub struct LoginJwtClaims {
    pub nbf: i64,
    pub exp: i64,
    pub iss: String,
    pub sub: UserId,      // ← 类型是 UserId，不是 String
    pub premium: bool,
    pub name: String,
    pub email: String,
    pub email_verified: bool,
    pub sstamp: String,   // security_stamp
    pub device: DeviceId,
    pub devicetype: String,
    pub client_id: String,
    pub scope: Vec<String>,
    pub amr: Vec<String>,
}
```

`claims.sub.to_string()` 将 `UserId` 转为字符串作为 `DashMap` 的 key。每个用户的 WebSocket 连接按 `UserId` 分组，同一用户可以有多个并发连接（多个设备）。

**注意：WebSocket 鉴权使用 access_token（`decode_login`），不是 refresh_token（`decode_refresh`）。** access_token 有效期短（默认 2 小时），且包含 `sstamp`（security_stamp）。但 WebSocket 连接建立后不再重新验证 token——如果用户修改密码导致 security_stamp 变更，已建立的 WebSocket 连接不会断开，只是后续的 HTTP API 调用会因 stamp 不匹配而失败。

### 5.3 匿名 WebSocket（anonymous-hub）

```rust
// api/notifications.rs
#[get("/anonymous-hub?<token..>")]
fn anonymous_websockets_hub<'r>(ws: WebSocket, token: String, ip: ClientIp) -> Result<rocket_ws::Stream!['r], Error> {
    let (tx, rx) = tokio::sync::mpsc::channel::<Message>(100);
    subscriptions.map.insert(token.clone(), tx);
    // ...
}
```

**匿名 WebSocket 的安全设计：**
- **无 JWT 验证**：`token` 直接来自 URL 参数，是 `String` 类型（非 Option），即必须提供
- **token 就是订阅 ID**：作为 `DashMap` 的 key，用于 AuthRequest 推送
- **token 生成**：登录流程创建 `AuthRequest` 时生成随机 UUID，存入数据库的 `auth_request` 表
- **推送时验证**：`push_auth_response` 通过数据库查询 token 对应的 `AuthRequest` 是否存在且有效
- **只能接收推送**：服务器端忽略除 Ping/Pong 和 INITIAL_MESSAGE 之外的所有客户端消息

### 5.4 WebSocket 与安全 Header 的协作

```
WebSocket 连接建立流程：

  HTTP GET /notifications/hub?access_token=xxx
       │
       ├─ 1. Rocket FromRequest 守卫执行
       │   ├─ ClientIp → 提取 IP（可失败，回退到 0.0.0.0）
       │   └─ WsAccessTokenHeader → 解析 Authorization Header（永远成功）
       │
       ├─ 2. 路由处理函数执行
       │   ├─ Token 优先级：URL 参数 → Header → 401
       │   ├─ decode_login → JWT 验证（可失败 → 401）
       │   └─ claims.sub(UserId) → 注册到 WS_USERS
       │
       ├─ 3. Response Fairing 执行（此时 JWT 验证结果已确定）
       │   ├─ AppHeaders → 检测 WebSocket 握手，跳过安全头
       │   └─ Cors → 根据 Origin 决定 Allow-Origin
       │
       └─ 4. 协议升级 → WebSocket 连接建立
```

**关键点：**
- `WsAccessTokenHeader` 是**非阻塞守卫**（永远 Success），与 `Headers` 守卫的阻塞行为不同
- JWT 验证在**路由处理函数**中，验证失败返回 401，不会触发安全 Header 例外
- 只有 JWT 验证成功后协议升级时，AppHeaders 才会跳过安全 Header
- WebSocket 连接建立后不再重新验证 token → security_stamp 变更不会断开已有连接

---

## 六、代理场景与 Header 信任边界

### 6.1 ClientIp 请求守卫（auth.rs）

```rust
pub struct ClientIp {
    pub ip: IpAddr,
}

impl<'r> FromRequest<'r> for ClientIp {
    async fn from_request(req: &'r Request<'_>) -> Outcome<Self, Self::Error> {
        let ip = if CONFIG._ip_header_enabled() {
            req.headers().get_one(&CONFIG.ip_header()).and_then(|ip| {
                match ip.find(',') {
                    Some(idx) => &ip[..idx],  // X-Forwarded-For: client, proxy1, proxy2
                    None => ip,
                }.parse().ok()
            })
        } else {
            None
        };

        let ip = ip.or_else(|| req.remote().map(|r| r.ip()))
            .unwrap_or_else(|| "0.0.0.0".parse().unwrap());

        Outcome::Success(ClientIp { ip })
    }
}
```

### 6.2 两套 IP 检测机制的差异

项目中存在**两个不同的 IP 来源检测结构**：

| 结构 | 用途 | IP 来源优先级 | 位置 |
|------|------|-------------|------|
| **ClientIp** | 限流、事件日志 | `CONFIG.ip_header` → `req.remote()` → `"0.0.0.0"` | auth.rs |
| **IpHeader** | 管理后台诊断显示 | `CONFIG.ip_header` → `X-Client-IP` → `X-Real-IP` → `X-Forwarded-For` → `None` | api/admin.rs |

**关键差异：**
- **ClientIp** 只信任管理员明确配置的一个 Header，用于安全决策（限流）
- **IpHeader** 依次探测多个 Header，仅用于管理后台诊断页展示信息

### 6.3 IP Header 配置与信任边界

| 配置项 | 默认值 | 生成逻辑 |
|--------|--------|---------|
| `ip_header` | `"X-Real-IP"` | 用户配置 |
| `_ip_header_enabled` | `true` | `ip_header.trim().to_lowercase() != "none"` |

**信任边界分析（四种场景）：**

| 场景 | 配置 | 信任来源 | 风险 |
|------|------|---------|------|
| **直接暴露** | `IP_HEADER=none` | TCP 连接 IP | 极低（TCP 层难以伪造） |
| **单层可信代理** | `IP_HEADER=X-Real-IP` | Nginx 设置的 Header | 低（需确保 Nginx 覆盖而非追加） |
| **多层代理/CDN** | `IP_HEADER=X-Forwarded-For` | Header 第一段（逗号前） | 中（最外层代理必须完全覆盖 Header） |
| **配置错误** | 设了 Header 但代理没发送 | 回退到 `req.remote()`（代理 IP） | 高（所有用户共享限流桶） |

**X-Forwarded-For 解析细节：**
- 取逗号前的第一段 IP：`"1.2.3.4, 5.6.7.8"` → `"1.2.3.4"`
- **安全前提**：最外层代理会覆盖而非追加这个 Header
- 如果外层代理是追加模式，攻击者可以注入 `X-Forwarded-For: fake-ip` 绕过限流

### 6.4 Secure 检测（auth.rs）

```rust
pub struct Secure {
    pub https: bool,
}
```

Secure 守卫信任 `X-Forwarded-Proto` 判断 HTTPS。此判断直接影响 Cookie 的 Secure 标记。

### 6.5 Host 守卫的代理信任（auth.rs）

```rust
pub struct Host {
    pub host: String,
}
```

Host 的信任策略是**配置优先**：只要设置了 `DOMAIN` 环境变量，就完全忽略请求头中的 Host/Forwarded 信息。

---

## 七、安全分层协作——完整请求处理链路

### 7.1 认证守卫分层

```
请求进入
  │
  ├─ ClientIp          → 提取 IP（用于限流和审计）
  ├─ Secure            → 检测 HTTPS（用于 Cookie 安全标记）
  │
  ├─ ClientHeaders     → ClientIp + device-type Header
  │   (用于 identity 登录路由，无需 Bearer Token)
  │
  ├─ Headers           → ClientIp + Host + JWT access_token 验证
  │   (用于需认证的 API 路由)
  │   验证链:
  │     1. 提取 Authorization: Bearer <token>
  │     2. decode_login → JWT 签名/过期/签发者 验证
  │     3. Device::find_by_uuid_and_user → 设备存在性验证
  │     4. User::find_by_uuid → 用户存在性验证
  │     5. security_stamp 匹配 → 安全戳校验
  │
  ├─ OrgHeaders        → Headers + 组织权限验证
  ├─ AdminHeaders      → JWT admin token 验证
  └─ ...
```

### 7.2 安全戳（Security Stamp）机制

Headers 守卫中的 `security_stamp` 校验是一个关键的安全层：用户密码修改、2FA 变更等敏感操作会更新 `security_stamp`，使所有已签发的 JWT 失效。

### 7.3 事件审计与限流的互补

无论登录成功或失败，都会记录 `log_user_event`，但只有 `user_id` 非空时才记录——即用户名存在并找到了对应 User。攻击者用随机用户名尝试不会产生审计记录，这意味着**限流是随机用户名攻击的唯一防线**。

### 7.4 完整协作流程图

```
┌───────────────────────────────────────────────────────────────┐
│                     HTTP Request                               │
└──────────────────┬────────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────────┐
│  第一层：Rocket 请求守卫 (FromRequest)                         │
│                                                                │
│  ClientIp ─── 从配置的 IP_HEADER 提取真实 IP                   │
│       │       回退到 req.remote() → "0.0.0.0"                  │
│       │                                                        │
│       ├─→ 限流桶查询（LIMITER_LOGIN / LIMITER_ADMIN）          │
│       │       ├─ 通过 → 继续                                   │
│       │       └─ 拒绝 → HTTP 429                               │
│       │                                                        │
│  Secure ───── 从 X-Forwarded-Proto 检测 HTTPS                  │
│       │       → 影响 Cookie Secure 标记                        │
│       │                                                        │
│  Host ─────── DOMAIN 配置优先，否则回退 Header 猜测            │
│                                                                │
└──────────────────┬────────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────────┐
│  第二层：认证守卫 (Headers / ClientHeaders)                     │
│                                                                │
│  未认证路由（login / register）：                               │
│    ClientHeaders → ClientIp + device-type                      │
│    → 限流检查 → 业务逻辑                                       │
│                                                                │
│  已认证路由（/api/*）：                                         │
│    Headers → JWT 验证 → Device 查询 → User 查询               │
│           → security_stamp 校验 → stamp_exception 兜底         │
│    → 业务逻辑                                                  │
│                                                                │
│  管理后台路由（/admin/*）：                                     │
│    AdminToken → JWT admin token 验证                           │
│    → 业务逻辑                                                  │
│                                                                │
│  WebSocket 路由（/notifications/hub）：                         │
│    WsAccessToken(URL) + WsAccessTokenHeader(Header)            │
│    → decode_login 验证 → 注册到 WS_USERS                       │
│                                                                │
└──────────────────┬────────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────────┐
│  第三层：事件审计                                               │
│                                                                │
│  log_user_event → 记录 UserLoggedIn / UserFailedLogIn 等       │
│  → 仅在 user_id 存在时记录（防信息泄露）                        │
│  → 限流是随机用户名攻击的唯一防线                               │
│                                                                │
└──────────────────┬────────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────────┐
│  第四层：响应安全 (Fairing)                                     │
│                                                                │
│  AppHeaders → 根据 URI/Content-Type 动态设置安全 Header        │
│    ├─ WebSocket 握手：跳过 X-Frame-Options 等三个 Header        │
│    ├─ connector.html：跳过 CSP + 移除 X-Frame-Options         │
│    ├─ 图片/icon_external：跳过 CORP，使用严格图片 CSP          │
│    └─ 其他：完整安全 Header                                    │
│                                                                │
│  Cors → 白名单 Origin 检查 + OPTIONS 预检响应                  │
│    ├─ Origin 匹配 → 设置 Allow-Origin                          │
│    ├─ Origin 不匹配 → 不设置 Allow-Origin（浏览器拦截）        │
│    └─ OPTIONS → 回显 Allow-Methods/Allow-Headers + 空响应      │
│                                                                │
└──────────────────┬────────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────────┐
│                     HTTP Response                              │
└───────────────────────────────────────────────────────────────┘
```

---

## 八、关键安全设计原则

### 8.1 分层防护与职责分离

| 层级 | 防护目标 | 实现机制 | 失败后果 |
|------|---------|---------|---------|
| **网络层** | DDoS、粗粒度攻击 | 反向代理/WAF | 大规模攻击直接穿透 |
| **应用限流层** | 暴力破解 | governor 令牌桶 | 单 IP 高频猜测密码 |
| **认证守卫层** | 凭据伪造/过期 | JWT + security_stamp | 旧凭据持续访问 |
| **业务审计层** | 事后追溯 | log_user_event | 无法发现异常登录模式 |
| **响应安全层** | 浏览器端攻击 | CSP/CORS/CORP | XSS/Clickjacking |

### 8.2 代理 Header 信任原则

1. **最小信任**：ClientIp 只信任管理员明确配置的一个 Header
2. **配置驱动**：`IP_HEADER=none` 可完全禁用代理 Header 信任
3. **Host 硬编码优先**：`DOMAIN` 设置后完全忽略请求头中的 Host 信息

### 8.3 例外规则的合理性

每个安全 Header 例外都有**明确的功能需求驱动**，且有**替代安全措施**：

| 例外 | 驱动需求 | 替代安全 |
|------|---------|---------|
| WebSocket 跳过三个 Header | 协议兼容性 | WebSocket 自身的 Origin 验证 + JWT 认证 |
| connector.html 跳过 CSP | MFA iframe | 仅影响特定静态文件 |
| 图片跳过 CORP | 桌面客户端兼容 | 图片专用严格 CSP（禁止 script/frame/object） |

### 8.4 CORS 的浏览器-centric 设计

- CORS 不阻止请求执行，只控制浏览器是否将响应暴露给 JavaScript
- OPTIONS 预检的 Allow-Headers/Allow-Methods 回显设计是兼容性优先
- Origin 白名单匹配是精确相等，不是通配符匹配

---

## 九、配置参考

### 限流配置
```env
LOGIN_RATELIMIT_SECONDS=60
LOGIN_RATELIMIT_MAX_BURST=10
ADMIN_RATELIMIT_SECONDS=300
ADMIN_RATELIMIT_MAX_BURST=3
```

### 代理配置
```env
# 单层可信代理（Nginx）
IP_HEADER=X-Real-IP

# 多层代理或 CDN（Cloudflare）
IP_HEADER=X-Forwarded-For

# 直接暴露（无代理）
IP_HEADER=none

# 必须设置 DOMAIN 以避免 Host Header 注入
DOMAIN=https://vault.example.com
```

### CSP 扩展配置
```env
ALLOWED_IFRAME_ANCESTORS=
ALLOWED_CONNECT_SRC=
ICON_SERVICE_CSP=
```

---

## 总结

Vaultwarden 的安全防护体系体现了"深度防御"理念，四层防护各有侧重：

1. **限流层**在入口处阻止暴力破解，覆盖密码/SSO/API Key/2FA 邮件发送四类入口，`refresh_login` 因 JWT 自身安全性而豁免限流
2. **认证守卫层**通过 JWT + security_stamp 确保已认证请求的凭据时效性，`refresh_tokens` 包含旧格式 Base64 token 的兼容逻辑——旧格式跳过 JWT 验证但必须通过数据库 Device 查询，且 `sub=Password` 硬编码会在 `sso_only` 模式下拒绝旧 token
3. **WebSocket 层**区分已认证（hub）和匿名（anonymous-hub）两种场景：前者使用 `decode_login` 验证 access_token（不是 refresh_token），后者依赖 token 的随机性和数据库验证；`WsAccessTokenHeader` 是非阻塞守卫（永远 Success），JWT 验证延迟到路由处理函数中执行
4. **安全 Headers 层**通过 CSP/CORS/CORP 等在浏览器端提供防护，WebSocket 握手、connector.html、图片等例外规则均有明确的功能需求和替代安全措施
5. **CORS 层**采用 Response Fairing 设计，在路由处理和数据库查询**之后**才执行，Origin 精确匹配白名单——不匹配时不设置 Allow-Origin 由浏览器拦截响应读取，但服务器端操作已执行完毕；CORS 防跨域数据泄露，与防跨域操作执行的 CSRF Token 互补
6. **代理支持层**通过 `IP_HEADER` 配置和 `DOMAIN` 硬编码确保在复杂部署环境下安全机制依然有效

各模块通过 Rocket 的 Fairing 和 FromRequest trait 实现解耦，但又在安全逻辑上形成完整的防护链。
