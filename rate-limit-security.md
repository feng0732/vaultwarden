# Vaultwarden 限流与安全防护协作机制

## 概述

Vaultwarden 通过多层次的安全防护机制实现完整的安全体系，包括：
- **请求限流**：基于 IP 的速率限制，防止暴力破解
- **登录限制**：多种登录方式的统一限流保护（含刷新令牌的例外设计）
- **安全 Headers**：CSP、CORS 等 Web 安全标准，含精细的例外规则
- **代理支持**：正确处理反向代理场景下的真实 IP，含信任边界分析
- **认证守卫分层**：从 JWT 验证到安全戳校验的多级防线

---

## 一、限流核心实现

### 1.1 限流模块结构 ([ratelimit.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/ratelimit.rs))

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

**配置位置：** [config.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/config.rs#L767-L774)

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

对整个 `src/api` 目录搜索 `ratelimit::check_limit`，确认限流的**实际覆盖范围**：

### 2.1 被限流保护的入口（5 处）

| # | 路由函数 | 限流器 | 代码位置 |
|---|---------|--------|---------|
| 1 | [password_login](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L359) | `check_limit_login` | identity.rs:359 |
| 2 | [sso_login](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L189) | `check_limit_login` | identity.rs:189 |
| 3 | [api_key_login](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L572) | `check_limit_login` | identity.rs:572 |
| 4 | [send_email_login](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/core/two_factor/email.rs#L48) | `check_limit_login` | email.rs:48 |
| 5 | [post_admin_login](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/admin.rs#L194) | `check_limit_admin` | admin.rs:194 |

### 2.2 未被限流保护的关键入口——refresh_login

[refresh_login](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L138-L176) 是 `grant_type="refresh_token"` 的处理函数。观察入口分发逻辑：

```rust
// identity.rs:70-73
"refresh_token" => {
    check_is_some(data.refresh_token.as_ref(), "refresh_token cannot be blank")?;
    refresh_login(data, &conn, &client_header.ip).await  // ← 无 check_limit_login
}
```

**refresh_login 不受限流保护的设计原因：**

1. **已有强认证保护**：`refresh_login` 调用 [auth::refresh_tokens](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L1248-L1300)，该函数必须：
   - 提供有效的 JWT refresh_token
   - 通过 `decode_refresh` 验证 JWT 签名、过期时间和签发者
   - 在数据库中查找匹配的 Device 记录（`Device::find_by_refresh_token`）
   - 关联到有效 User

2. **令牌本身即限流器**：refresh_token 是一个有有效期的 JWT（默认 30 天，移动端 90 天），且绑定到特定设备。攻击者需要先窃取有效的 refresh_token 才能发起请求，无法像密码登录那样随意尝试。

3. **避免影响正常用户体验**：已认证用户的 token 刷新是高频操作（access_token 默认 2 小时过期），限流会严重影响使用流畅度。

4. **失败成本高**：无效的 refresh_token 会在 JWT 解码阶段就被拒绝，不会触发任何数据库查询，对服务器压力极小。

```
┌─────────────────────────────────────────────────────────────────┐
│  grant_type 分发 (login 函数)                                    │
│                                                                   │
│  "password"          → check_limit_login → password_login        │
│  "authorization_code"→ check_limit_login → sso_login             │
│  "client_credentials"→ check_limit_login → api_key_login         │
│  "refresh_token"     → (无限流)          → refresh_login  ★      │
│                                                                   │
│  ★ refresh_login 安全保障链:                                      │
│    JWT 签名验证 → JWT 过期检查 → Device 查询 → User 查询          │
│    任意环节失败即返回 invalid_grant (400)                          │
└─────────────────────────────────────────────────────────────────┘
```

**潜在风险与缓解：** 如果 refresh_token 泄露，攻击者可以在有效期内无限刷新。缓解措施包括：
- refresh_token 绑定设备 UUID，换设备无法使用
- 用户或管理员可以通过清除设备记录使 token 失效
- SSO 模式下，`sso_auth_only_not_session()` 配置可强制 SSO 会话验证

---

## 三、安全 Headers 防护——例外规则详解

### 3.1 AppHeaders Fairing ([util.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/util.rs#L28-L148))

`AppHeaders` 是一个 Rocket Response Fairing，在**每个响应**上设置安全头。其判断逻辑有三层例外：

### 3.2 例外场景 1：WebSocket 连接

```rust
// util.rs:43-58
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

**注意细节**：不是所有对 `/notifications/hub` 的请求都跳过安全头，只有同时携带 `Connection: Upgrade` + `Upgrade: websocket` 的**真正的 WebSocket 握手请求**才跳过。普通的 HTTP 请求到同一路径仍会设置安全 Header。

### 3.3 例外场景 2：MFA connector.html

```rust
// util.rs:82-84
if req_uri_path.ends_with("connector.html") {
    res.remove_header("X-Frame-Options");  // 仅移除 X-Frame-Options
    // 不设置 CSP
} else {
    // 正常设置 CSP + X-Frame-Options
}
```

**原因**：WebAuthn 和 Duo 等 MFA 方案需要通过 iframe 弹窗完成验证，CSP 和 X-Frame-Options 会阻断这一流程。这与上游 Bitwarden 行为一致。

### 3.4 例外场景 3：图片与 icon_external 路由

```rust
// util.rs:69-77
let mut is_image = true;
if !(res.headers().get_one("Content-Type").is_some_and(|v| v.starts_with("image/"))
    || req.route().is_some_and(|v| v.name.as_deref() == Some("icon_external")))
{
    is_image = false;
    res.set_raw_header("Cross-Origin-Resource-Policy", "same-origin");
}
```

两种情况不设置 `Cross-Origin-Resource-Policy`：
1. **Content-Type 为 image/** 的响应（如 SVG 图标）
2. **路由名为 `icon_external`** 的请求（[icons.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/icons.rs#L82-L109)）

**原因**：Bitwarden 桌面客户端需要跨域下载图标，设置 `CORP: same-origin` 会导致图标加载失败。`icon_external` 函数名的硬编码关联在代码注释中有明确说明：

```rust
// icons.rs:82-84
// The function name `icon_external` is checked in the `on_response` function in `AppHeaders`
// It is used to prevent sending a specific header which breaks icon downloads.
// If this function needs to be renamed, also adjust the code in `util.rs`
```

图片类型使用独立的**严格 CSP**（禁止所有 script/frame/object）来弥补 CORP 缺失带来的风险，特别是防止 SVG 中的 JavaScript 执行。

### 3.5 安全 Header 例外规则汇总

| 场景 | 跳过的 Header | 保留的 Header | 安全原因 |
|------|--------------|--------------|---------|
| WebSocket 握手 | X-Frame-Options, X-Content-Type-Options, Permissions-Policy, CSP, CORP, Referrer-Policy 等 | 无 | 协议兼容性（反向代理/Cloudflare） |
| `*-connector.html` | X-Frame-Options, CSP | 其余全部保留 | MFA iframe 需要 |
| `Content-Type: image/*` | Cross-Origin-Resource-Policy | 其余全部保留 + 图片专用 CSP | 桌面客户端图标下载兼容 |
| `icon_external` 路由 | Cross-Origin-Resource-Policy | 其余全部保留 + 图片专用 CSP | 桌面客户端图标下载兼容 |

### 3.6 管理后台诊断页自检

[admin_diagnostics.js](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/static/scripts/admin_diagnostics.js#L242-L317) 中的 `checkSecurityHeaders` 函数会自动检测安全 Header 是否正确设置：

- **API 调用**：检查所有标准安全 Header
- **2FA Connector**：验证 `x-frame-options` 和 `content-security-policy` **不应出现**
- **HTTP 错误响应**（404/400/401/403）：验证不被反向代理覆盖

如果反向代理重写了响应头，诊断页会显示警告。这形成了一个**自愈闭环**：代码设置 Header → 诊断页检测 → 管理员发现并修复代理配置。

---

## 四、CORS 跨域配置 ([util.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/util.rs#L150-L208))

### 4.1 Cors Fairing

```rust
pub struct Cors();

impl Cors {
    fn get_allowed_origin(headers: &HeaderMap<'_>) -> Option<String> {
        let origin = Cors::get_header(headers, "Origin");

        if origin == CONFIG.domain_origin()       // 配置的域名
            || origin == safari_extension_origin    // "file://"
            || origin == desktop_custom_file_origin // "bw-desktop-file://bundle"
            || (CONFIG.sso_enabled() && origin == CONFIG.sso_authority())
        {
            Some(origin)
        } else {
            None
        }
    }
}
```

### 4.2 预检请求处理

对于 OPTIONS 请求：
```rust
if request.method() == Method::Options {
    response.set_header(Header::new("Access-Control-Allow-Methods", req_allow_method));
    response.set_header(Header::new("Access-Control-Allow-Headers", req_allow_headers));
    response.set_header(Header::new("Access-Control-Allow-Credentials", "true"));
    response.set_status(Status::Ok);
}
```

**CORS 与安全 Header 的协作**：CORS Fairing 和 AppHeaders Fairing 是**独立的** Response Fairing，两者都会对同一个响应生效。Cors 控制 `Access-Control-*` 系列头，AppHeaders 控制 `Content-Security-Policy`、`X-Frame-Options` 等。CORS 的白名单机制确保只有已知客户端来源能获得跨域许可，而 CSP 的 `frame-ancestors` 和 `connect-src` 进一步在内容层面限制加载行为。

---

## 五、代理场景与 Header 信任边界

### 5.1 ClientIp 请求守卫 ([auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L1034-L1067))

```rust
#[derive(Copy, Clone)]
pub struct ClientIp {
    pub ip: IpAddr,
}

#[rocket::async_trait]
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

### 5.2 两套 IP 检测机制的差异

项目中存在**两个不同的 IP 来源检测结构**：

| 结构 | 用途 | IP 来源优先级 | 代码位置 |
|------|------|-------------|---------|
| **ClientIp** | 限流、事件日志 | `CONFIG.ip_header` → `req.remote()` → `"0.0.0.0"` | [auth.rs:1034-1067](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L1034-L1067) |
| **IpHeader** | 管理后台诊断显示 | `CONFIG.ip_header` → `X-Client-IP` → `X-Real-IP` → `X-Forwarded-For` → `None` | [admin.rs:118-138](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/admin.rs#L118-L138) |

**关键差异**：
- **ClientIp** 只信任 `CONFIG.ip_header` 配置的那一个 Header，用于安全决策（限流）
- **IpHeader** 依次探测多个 Header，仅用于管理后台诊断页展示信息，不参与安全决策

这种分离是**有意的安全设计**：限流只信任管理员明确配置的单一 Header，而诊断页需要尽可能发现客户端 IP 的来源信息以帮助排查问题。

### 5.3 IP Header 配置与信任边界

| 配置项 | 默认值 | 生成逻辑 | 代码位置 |
|--------|--------|---------|---------|
| `ip_header` | `"X-Real-IP"` | 用户配置 | [config.rs:666](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/config.rs#L666) |
| `_ip_header_enabled` | `true` | `ip_header.trim().to_lowercase() != "none"` | [config.rs:668](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/config.rs#L668) |

**信任边界分析：**

```
┌──────────────────────────────────────────────────────────────────┐
│                     代理 Header 信任模型                          │
│                                                                    │
│  场景 A：直接暴露（无代理）                                        │
│  IP_HEADER=none                                                    │
│  → _ip_header_enabled=false                                        │
│  → ClientIp 使用 req.remote()（TCP 连接 IP）                      │
│  → 信任边界：TCP 层，无法伪造 ✓                                    │
│                                                                    │
│  场景 B：单层可信代理（Nginx）                                     │
│  IP_HEADER=X-Real-IP                                               │
│  → ClientIp 读取 X-Real-IP                                        │
│  → 信任边界：Nginx 配置（必须确保未认证请求无法设置该 Header）      │
│  → Nginx 应使用 proxy_set_header X-Real-IP $remote_addr           │
│                                                                    │
│  场景 C：多层代理或 CDN（Cloudflare）                              │
│  IP_HEADER=X-Forwarded-For                                         │
│  → ClientIp 读取 X-Forwarded-For 的第一段（逗号前）               │
│  → 信任边界：最外层代理覆盖整个 X-Forwarded-For                   │
│  → ⚠️ 风险：中间代理未清除客户端注入的 X-Forwarded-For            │
│                                                                    │
│  场景 D：IP_HEADER 设置但代理未发送该 Header                       │
│  → ClientIp 回退到 req.remote()（代理 IP）                        │
│  → 所有用户共享代理 IP → 限流桶合并 → 可能误触发                   │
└──────────────────────────────────────────────────────────────────┘
```

**X-Forwarded-For 解析细节**：

```rust
// auth.rs:1049-1053
match ip.find(',') {
    Some(idx) => &ip[..idx],  // "1.2.3.4, 5.6.7.8" → 取 "1.2.3.4"
    None => ip,               // "1.2.3.4" → 原样使用
}
```

取逗号前的第一段 IP，对应 X-Forwarded-For 格式中"最左边的客户端 IP"。**安全前提**是最外层代理会覆盖而非追加这个 Header。如果外层代理是追加模式，攻击者可以在请求中注入 `X-Forwarded-For: fake-ip` 来伪造 IP，绕过限流。

### 5.4 Secure 检测 ([auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L1069-L1097))

```rust
pub struct Secure {
    pub https: bool,
}
```

Secure 守卫信任 `X-Forwarded-Proto` 判断 HTTPS。此判断直接影响 **Cookie 的 Secure 标记**：
- 管理后台 Cookie（[admin.rs:212](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/admin.rs#L212)）：`.secure(secure.https)`
- SSO 绑定 Cookie（[identity.rs:1301](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L1301)）：`.secure(secure.https)`

如果 `X-Forwarded-Proto` 被伪造为 `https`，Cookie 的 Secure 标记会在 HTTP 连接上设置，但浏览器实际上不会在纯 HTTP 请求中发送 Secure Cookie，因此风险可控。

### 5.5 Host 守卫的代理信任 ([auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L540-L574))

```rust
pub struct Host {
    pub host: String,
}

async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error> {
    let host = if CONFIG.domain_set() {
        CONFIG.domain()   // ← 优先使用配置的 DOMAIN，不信任 Header
    } else if let Some(referer) = headers.get_one("Referer") {
        referer.to_owned()
    } else {
        let protocol = headers.get_one("X-Forwarded-Proto").unwrap_or_else(|| ...);
        let host = headers.get_one("X-Forwarded-Host").unwrap_or_else(|| headers.get_one("Host"));
        format!("{protocol}://{host}")
    };
}
```

Host 的信任策略是**配置优先**：只要设置了 `DOMAIN` 环境变量，就完全忽略请求头中的 Host/Forwarded 信息。只有未配置 DOMAIN 时才回退到 Header 猜测。这是一个**防御性设计**，避免 Host Header 注入攻击。

---

## 六、安全分层协作——完整请求处理链路

### 6.1 认证守卫分层

Vaultwarden 定义了多层认证守卫，每层递进验证：

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
  │     5. security_stamp 匹配 → ★ 安全戳校验
  │        └─ 不匹配时检查 stamp_exception（特定路由 + 过期时间）
  │
  ├─ OrgHeaders        → Headers + 组织权限验证
  ├─ AdminHeaders      → JWT admin token 验证
  └─ ...
```

### 6.2 安全戳（Security Stamp）机制

[Headers 守卫](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L653-L681) 中的 `security_stamp` 校验是一个关键的安全层：

```rust
if user.security_stamp != claims.sstamp {
    if let Some(stamp_exception) = ... {
        // 检查例外是否过期
        if Utc::now().timestamp() > stamp_exception.expire { ... }
        // 检查当前路由是否在允许列表中
        if !stamp_exception.routes.contains(&current_route) { ... }
        // 检查例外中的 stamp 是否匹配
        if stamp_exception.security_stamp != claims.sstamp { ... }
    } else {
        err_handler!("Invalid security stamp")
    }
}
```

**工作原理**：用户密码修改、2FA 变更等敏感操作会更新 `security_stamp`，使所有已签发的 JWT 失效（因为 JWT 中嵌入的 sstamp 不再匹配）。`stamp_exception` 机制允许在 stamp 变更后短时间内，特定路由（如密码修改本身）仍可使用旧 stamp，避免操作中断。

**与限流的协作**：限流在登录入口阻止暴力破解；安全戳在已认证 API 调用中确保凭据时效性。两者分别保护"进入"和"持续访问"两个阶段。

### 6.3 事件审计与限流的互补

[identity.rs:114-133](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L114-L133) 中，无论登录成功或失败，都会记录 `log_user_event`：

```rust
if let Some(user_id) = user_id {
    match &login_result {
        Ok(_) => log_user_event(EventType::UserLoggedIn, ...),
        Err(e) => {
            if let Some(ev) = e.get_event() {
                log_user_event(ev.event, ...);  // 如 UserFailedLogIn, UserFailedLogIn2fa
            }
        }
    }
}
```

**注意**：只有 `user_id` 非空时才记录事件——即用户名存在并找到了对应 User。攻击者用随机用户名尝试不会产生审计记录，这是**信息泄露防护**（不确认用户名是否存在），但也意味着限流是唯一的防御手段。

### 6.4 完整协作流程图

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
│    ├─ WebSocket 握手：跳过全部                                 │
│    ├─ connector.html：跳过 CSP + X-Frame-Options              │
│    ├─ 图片/icon_external：跳过 CORP，使用严格图片 CSP          │
│    └─ 其他：完整安全 Header                                    │
│                                                                │
│  Cors → 白名单 Origin 检查 + OPTIONS 预检                     │
│                                                                │
│  诊断自检 → admin_diagnostics.js 验证安全 Header 正确性        │
│                                                                │
└──────────────────┬────────────────────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────────────────────┐
│                     HTTP Response                              │
└───────────────────────────────────────────────────────────────┘
```

---

## 七、关键安全设计原则

### 7.1 分层防护与职责分离

| 层级 | 防护目标 | 实现机制 | 失败后果 |
|------|---------|---------|---------|
| **网络层** | DDoS、粗粒度攻击 | 反向代理/WAF | 大规模攻击直接穿透 |
| **应用限流层** | 暴力破解 | governor 令牌桶 | 单 IP 高频猜测密码 |
| **认证守卫层** | 凭据伪造/过期 | JWT + security_stamp | 旧凭据持续访问 |
| **业务审计层** | 事后追溯 | log_user_event | 无法发现异常登录模式 |
| **响应安全层** | 浏览器端攻击 | CSP/CORS/CORP | XSS/Clickjacking |

每一层独立运作，不依赖其他层。即使某层被绕过，其他层仍提供保护。

### 7.2 代理 Header 信任原则

1. **最小信任**：ClientIp 只信任管理员明确配置的一个 Header，不像 IpHeader 那样依次探测
2. **配置驱动**：`_ip_header_enabled` 由配置自动生成，`IP_HEADER=none` 可完全禁用代理 Header 信任
3. **Host 硬编码优先**：`DOMAIN` 设置后完全忽略请求头中的 Host 信息
4. **Secure 标记可控**：`X-Forwarded-Proto` 仅影响 Cookie 的 Secure 标记，浏览器层面的保护限制了伪造风险

### 7.3 限流粒度权衡

当前实现：**按 IP 限流**
- ✅ 优点：简单有效，防止单 IP 暴力破解
- ⚠️ 不足：NAT 环境下多用户共享 IP 可能互相影响
- 注意：`refresh_login` 不受限流保护，依赖 JWT 自身的安全性

### 7.4 例外规则的合理性

每个安全 Header 例外都有**明确的功能需求驱动**，且有**替代安全措施**：

| 例外 | 驱动需求 | 替代安全 |
|------|---------|---------|
| WebSocket 跳过全部 Header | 协议兼容性 | WebSocket 自身的 origin 验证 |
| connector.html 跳过 CSP | MFA iframe | 仅影响特定静态文件 |
| 图片跳过 CORP | 桌面客户端兼容 | 图片专用严格 CSP（禁止 script/frame/object） |

---

## 八、配置参考

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
2. **认证守卫层**通过 JWT + security_stamp 确保已认证请求的凭据时效性，与限流分别保护"进入"和"持续访问"
3. **安全 Headers 层**通过 CSP/CORS/CORP 等在浏览器端提供防护，例外规则（WebSocket/connector.html/图片）均有明确的功能需求和替代安全措施
4. **代理支持层**通过 `IP_HEADER` 配置和 `DOMAIN` 硬编码确保在复杂部署环境下安全机制依然有效，ClientIp 和 IpHeader 的分离体现了"安全决策最小信任、诊断信息最大发现"的原则

各模块通过 Rocket 的 Fairing 和 FromRequest trait 实现解耦，但又在安全逻辑上形成完整的防护链。管理后台诊断页的自检机制形成了安全闭环——代码设置 Header → 诊断页检测 → 管理员发现并修复配置问题。
