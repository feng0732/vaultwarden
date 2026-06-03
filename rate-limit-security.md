# Vaultwarden 限流与安全防护协作机制

## 概述

Vaultwarden 通过多层次的安全防护机制实现完整的安全体系，包括：
- **请求限流**：基于 IP 的速率限制，防止暴力破解
- **登录限制**：多种登录方式的统一限流保护
- **安全 Headers**：CSP、CORS 等 Web 安全标准
- **代理支持**：正确处理反向代理场景下的真实 IP

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

### 1.2 限流检查函数

```rust
// 登录限流检查 - 应用于所有登录相关接口
pub fn check_limit_login(ip: &IpAddr) -> Result<(), Error>

// 管理后台限流检查
pub fn check_limit_admin(ip: &IpAddr) -> Result<(), Error>
```

触发限流时返回 HTTP 429 状态码：
```rust
err_code!("Too many login requests", 429);
```

---

## 二、登录限制场景

限流在所有登录入口处统一应用，确保全路径覆盖。

### 2.1 密码登录 ([identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L348-L360))

```rust
async fn password_login(...) -> JsonResult {
    // 1. 验证 scope
    AuthMethod::Password.check_scope(data.scope.as_ref())?;
    
    // 2. 限流检查（最外层，优先执行）
    crate::ratelimit::check_limit_login(&ip.ip)?;
    
    // 3. 查找用户、验证密码...
}
```

### 2.2 SSO 登录 ([identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L178-L190))

```rust
async fn sso_login(...) -> JsonResult {
    AuthMethod::Sso.check_scope(data.scope.as_ref())?;
    crate::ratelimit::check_limit_login(&ip.ip)?;  // 同样限流保护
    // SSO 交换代码流程...
}
```

### 2.3 API Key 登录 ([identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/identity.rs#L570-L580))

```rust
async fn api_key_login(...) -> JsonResult {
    crate::ratelimit::check_limit_login(&ip.ip)?;  // 同样限流保护
    // API Key 验证流程...
}
```

### 2.4 邮箱 2FA 发送 ([email.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/core/two_factor/email.rs#L39-L49))

```rust
async fn send_email_login(...) -> EmptyResult {
    crate::ratelimit::check_limit_login(&client_headers.ip.ip)?;
    // 发送邮箱验证码...
}
```

### 2.5 管理后台登录 ([admin.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/api/admin.rs#L184-L199))

```rust
fn post_admin_login(...) -> Result<Redirect, AdminResponse> {
    if crate::ratelimit::check_limit_admin(&ip.ip).is_err() {
        return Err(AdminResponse::TooManyRequests(...));
    }
    // 验证 Admin Token...
}
```

**关键设计点：**
- 限流检查在**所有业务逻辑之前**执行
- 登录相关接口共享**同一限流桶**（密码登录、SSO、API Key、2FA 发送）
- 管理后台使用**独立限流桶**
- 限流粒度为 **IP 地址**

---

## 三、安全 Headers 防护

### 3.1 AppHeaders Fairing ([util.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/util.rs#L28-L148))

`AppHeaders` 是一个 Rocket Fairing，在每个响应上设置安全头。

```rust
pub struct AppHeaders();

#[rocket::async_trait]
impl Fairing for AppHeaders {
    fn info(&self) -> Info {
        Info {
            name: "Application Headers",
            kind: Kind::Response,
        }
    }

    async fn on_response<'r>(&self, req: &'r Request<'_>, res: &mut Response<'r>) {
        // ... 安全 header 设置逻辑
    }
}
```

### 3.2 安全 Header 详解

| Header | 值 | 作用 |
|--------|----|------|
| **Permissions-Policy** | 全禁用 | 禁止所有浏览器敏感 API（摄像头、麦克风、定位等） |
| **Referrer-Policy** | `same-origin` | 仅同源请求发送 Referrer |
| **X-Content-Type-Options** | `nosniff` | 阻止 MIME 类型嗅探 |
| **X-Robots-Tag** | `noindex, nofollow` | 禁止搜索引擎索引 |
| **X-XSS-Protection** | `0` | 禁用浏览器 XSS 保护（被 CSP 取代） |
| **Cross-Origin-Resource-Policy** | `same-origin` | 阻止跨域资源加载（图片除外） |
| **Content-Security-Policy** | 动态生成 | 内容安全策略 |
| **X-Frame-Options** | `SAMEORIGIN` | 防止 Clickjacking |

### 3.3 CSP 策略分级

**1. 图片专用 CSP（防止 SVG XSS）**
```
default-src 'none';
img-src 'self' data:;
style-src 'unsafe-inline';
script-src 'none';
frame-src 'none';
object-src 'none'
```

**2. 普通页面 CSP**
```
default-src 'none';
font-src 'self';
script-src 'self' 'wasm-unsafe-eval';
style-src 'self' 'unsafe-inline';
frame-ancestors 'self' [扩展ID];
img-src 'self' data: https://haveibeenpwned.com [图标服务];
connect-src 'self' https://api.pwnedpasswords.com ...;
```

**例外情况：**
- WebSocket 连接路径（`/notifications/hub`）**不设置**安全 Header
- MFA connector.html 文件**不设置** CSP 和 X-Frame-Options
- 图片资源**不设置** Cross-Origin-Resource-Policy

---

## 四、CORS 跨域配置 ([util.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/util.rs#L150-L208))

### 4.1 Cors Fairing

```rust
pub struct Cors();

impl Cors {
    fn get_allowed_origin(headers: &HeaderMap<'_>) -> Option<String> {
        let origin = Cors::get_header(headers, "Origin");
        
        // 允许的来源：
        // 1. 配置的 domain_origin
        // 2. Safari 扩展 (file://)
        // 3. 桌面应用 (bw-desktop-file://bundle)
        // 4. SSO 授权地址（启用时）
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

---

## 五、代理场景与真实 IP 检测

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
            // 1. 从配置的 IP Header 读取
            req.headers().get_one(&CONFIG.ip_header()).and_then(|ip| {
                // 处理 X-Forwarded-For 格式（取第一个 IP）
                match ip.find(',') {
                    Some(idx) => &ip[..idx],
                    None => ip,
                }.parse().ok()
            })
        } else {
            None
        };

        // 2. 回退到 remote 地址
        let ip = ip.or_else(|| req.remote().map(|r| r.ip()))
            .unwrap_or_else(|| "0.0.0.0".parse().unwrap());

        Outcome::Success(ClientIp { ip })
    }
}
```

### 5.2 IP Header 配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `ip_header` | `X-Real-IP` | 读取真实 IP 的 Header 名称 |
| `_ip_header_enabled` | 自动 | 值为 "none" 时禁用 |

**代理配置示例：**
```env
# Nginx 代理
IP_HEADER=X-Real-IP

# Cloudflare 或多层代理
IP_HEADER=X-Forwarded-For

# 禁用（直接暴露时）
IP_HEADER=none
```

### 5.3 Secure 检测 ([auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/15-vaultwarden/src/auth.rs#L1069-L1097))

检测请求是否通过 HTTPS：

```rust
pub struct Secure {
    pub https: bool,
}

async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error> {
    let protocol = match headers.get_one("X-Forwarded-Proto") {
        Some(proto) => proto,
        None => {
            if env::var("ROCKET_TLS").is_ok() { "https" } else { "http" }
        }
    };
    Outcome::Success(Secure { https: protocol == "https" })
}
```

---

## 六、协作流程总图

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP Request                              │
└─────────────┬───────────────────────────────────────────────┘
              │
┌─────────────▼───────────────────────────────────────────────┐
│           Rocket 框架层                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. ClientIp 提取（代理 Header → 真实 IP）           │    │
│  │  2. Secure 检测（X-Forwarded-Proto → HTTPS）         │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────┬───────────────────────────────────────────────┘
              │
┌─────────────▼───────────────────────────────────────────────┐
│                    路由处理层                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  登录类路由 (/identity/*)                            │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │  限流检查 check_limit_login(ip)               │  │    │
│  │  │  → 通过：继续                                  │  │    │
│  │  │  → 拒绝：HTTP 429 Too Many Requests           │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  │                                                      │    │
│  │  管理后台路由 (/admin/*)                             │    │
│  │  ┌───────────────────────────────────────────────┐  │    │
│  │  │  限流检查 check_limit_admin(ip)               │  │    │
│  │  └───────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────┬───────────────────────────────────────────────┘
              │
┌─────────────▼───────────────────────────────────────────────┐
│              响应层 (Fairing)                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  AppHeaders:                                        │    │
│  │  - CSP (根据内容类型动态选择)                        │    │
│  │  - Permissions-Policy                                │    │
│  │  - X-Frame-Options / X-Content-Type-Options         │    │
│  │  - Referrer-Policy                                  │    │
│  │  - Cross-Origin-Resource-Policy (非图片)            │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Cors:                                              │    │
│  │  - Access-Control-Allow-Origin (白名单)             │    │
│  │  - OPTIONS 预检响应                                 │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────┬───────────────────────────────────────────────┘
              │
┌─────────────▼───────────────────────────────────────────────┐
│                    HTTP Response                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、关键安全设计原则

### 7.1 分层防护
1. **网络层**：反向代理限流 + WAF（可选）
2. **应用层**：governor 令牌桶限流
3. **业务层**：登录失败日志、账户锁定（事件记录）
4. **传输层**：安全 Headers、CSP

### 7.2 正确的代理配置重要性

**错误配置风险：**
- 如果未正确配置 `IP_HEADER`，限流将基于代理 IP 生效，导致：
  - 所有用户共享同一限流桶
  - 恶意用户可绕过限流
  - 正常用户可能被误拦截

**推荐配置：**
```env
# 单一层代理（Nginx）
IP_HEADER=X-Real-IP

# 多层代理或 CDN
IP_HEADER=X-Forwarded-For
```

### 7.3 限流粒度权衡

当前实现：**按 IP 限流**
- ✅ 优点：简单有效，防止单 IP 暴力破解
- ⚠️ 不足：NAT 环境下多用户共享 IP 可能互相影响
- 改进方向：可考虑按用户名 + IP 组合限流

### 7.4 WebSocket 例外

`/notifications/hub` 路径不设置安全 Header 的原因：
- WebSocket 协议升级机制与 HTTP Headers 存在冲突
- 部分反向代理和 Cloudflare 对 WebSocket 的 Header 处理有特殊要求
- 移除相关 Header 确保兼容性

---

## 八、配置参考

### 限流配置
```env
# 登录限流：每60秒平均1次请求，允许突发10次
LOGIN_RATELIMIT_SECONDS=60
LOGIN_RATELIMIT_MAX_BURST=10

# 管理后台限流：每300秒平均1次请求，允许突发3次
ADMIN_RATELIMIT_SECONDS=300
ADMIN_RATELIMIT_MAX_BURST=3
```

### 代理配置
```env
# 真实 IP Header
IP_HEADER=X-Real-IP

# 域名配置（影响 CSP 和 CORS）
DOMAIN=https://vault.example.com
DOMAIN_PATH=/
```

### CSP 扩展配置
```env
# 允许的 iframe 父域
ALLOWED_IFRAME_ANCESTORS=

# 额外允许的 connect-src
ALLOWED_CONNECT_SRC=

# 图标服务 CSP
ICON_SERVICE_CSP=
```

---

## 总结

Vaultwarden 的安全防护体系体现了"深度防御"理念：
1. **限流**在入口处阻止暴力破解攻击
2. **安全 Headers**在浏览器端提供全方位保护
3. **CORS**严格控制跨域访问
4. **代理支持**确保在复杂部署环境下安全机制依然有效

各模块之间通过 Rocket 的 Fairing 和 FromRequest trait 实现解耦，但又在安全逻辑上形成完整的防护链。
