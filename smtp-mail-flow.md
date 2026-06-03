# SMTP 邮件通知流程详解

本文档详细解析 Vaultwarden 中的邮件通知系统，包括事件触发、模板字段、发送流程和失败处理机制。

---

## 1. 核心架构概览

```
事件触发 → 模板渲染 → 邮件构建 → 传输发送 → 结果处理
    ↓          ↓          ↓          ↓          ↓
  API/定时   Handlebars  lettre    SMTP/Sendmail  错误分类
```

### 核心模块

| 模块 | 文件 | 职责 |
|------|------|------|
| 邮件核心发送 | [mail.rs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs) | 模板渲染、邮件构建、传输发送、错误处理 |
| SMTP 配置 | [config.rs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/config.rs#L862-L904) | SMTP 配置定义、验证、默认值处理 |
| 邮件模板 | [templates/email/](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email) | Handlebars 模板文件 |
| 事件触发 | 多个 API 模块 | 业务逻辑触发邮件发送 |

---

## 2. SMTP 配置详解

### 2.1 配置项定义 ([config.rs#L862-L904](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/config.rs#L862-L904))

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `_enable_smtp` | bool | `true` | SMTP 功能总开关 |
| `use_sendmail` | bool | `false` | 是否使用 sendmail 命令而非 SMTP |
| `sendmail_command` | String | - | sendmail 命令路径（可选） |
| `smtp_host` | String | - | SMTP 服务器地址 |
| `smtp_security` | String | `starttls` | 安全模式: `starttls`, `force_tls`, `off` |
| `smtp_port` | u16 | 自动 | 端口: force_tls→465, starttls→587, off→25 |
| `smtp_from` | String | - | 发件人地址 |
| `smtp_from_name` | String | `Vaultwarden` | 发件人显示名称 |
| `smtp_username` | String | - | SMTP 认证用户名 |
| `smtp_password` | Pass | - | SMTP 认证密码 |
| `smtp_auth_mechanism` | String | - | 认证机制: Plain, Login, Xoauth2 (逗号分隔) |
| `smtp_timeout` | u64 | `15` | 连接超时（秒） |
| `helo_name` | String | - | HELO 命令发送的主机名 |
| `smtp_embed_images` | bool | `true` | 是否将图片作为附件嵌入 |
| `smtp_accept_invalid_certs` | bool | `false` | 接受无效证书（危险） |
| `smtp_accept_invalid_hostnames` | bool | `false` | 接受无效主机名（危险） |

### 2.2 配置验证 ([config.rs#L1104-L1164](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/config.rs#L1104-L1164))

- 验证 `smtp_security` 必须是 `off`, `starttls`, `force_tls` 之一
- 使用 sendmail 时验证命令存在且可执行
- 使用 SMTP 时验证 `smtp_host` 和 `smtp_from` 必须同时设置
- 用户名和密码必须同时设置或同时不设置
- 验证 `smtp_from` 是有效的邮件地址
- 邮件 2FA 启用时必须配置邮件传输

### 2.3 邮件启用判断 ([config.rs#L1568-L1571](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/config.rs#L1568-L1571))

```rust
pub fn mail_enabled(&self) -> bool {
    let inner = &self.inner.read().unwrap().config;
    inner._enable_smtp && (inner.smtp_host.is_some() || inner.use_sendmail)
}
```

---

## 3. 事件触发机制

邮件通知由多种事件触发，主要分为以下几类：

### 3.1 用户账户相关

| 触发事件 | 触发位置 | 邮件函数 |
|----------|----------|----------|
| 用户注册（需验证） | [accounts.rs#L320](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L320) | `send_welcome_must_verify` |
| 用户注册（无需验证） | [accounts.rs#L324](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L324) | `send_welcome` |
| 邮箱验证请求 | [accounts.rs#L1065](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1065) | `send_verify_email` |
| 注册验证邮件 | [identity.rs#L1072](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L1072) | `send_register_verify_email` |
| 账户删除请求 | [accounts.rs#L1115](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1115) | `send_delete_account` |
| 密码提示 | [accounts.rs#L1212](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1212) | `send_password_hint` |
| 新设备登录 | [identity.rs#L480](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L480) | `send_new_device_logged_in` |
| 邮箱变更 | [accounts.rs#L962-L982](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L962-L982) | `send_change_email*` |
| SSO 邮箱变更 | [identity.rs#L330](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L330) | `send_sso_change_email` |

### 3.2 两步验证相关

| 触发事件 | 触发位置 | 邮件函数 |
|----------|----------|----------|
| 发送 2FA 令牌 | [email.rs#L120](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/email.rs#L120) | `send_token` |
| 2FA 配置验证邮件 | [email.rs#L188](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/email.rs#L188) | `send_token` |
| 组织移除 2FA | [two_factor/mod.rs#L186](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/mod.rs#L186) | `send_2fa_removed_from_org` |
| 受保护操作令牌 | [protected_actions.rs#L94](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/protected_actions.rs#L94) | `send_protected_action_token` |

### 3.3 组织管理相关

| 触发事件 | 触发位置 | 邮件函数 |
|----------|----------|----------|
| 组织邀请 | [organizations.rs#L1113](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1113) | `send_invite` |
| 邀请被接受 | [core/mod.rs#L296](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/mod.rs#L296) | `send_invite_accepted` |
| 邀请被确认 | [organizations.rs#L1320](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1320) | `send_invite_confirmed` |
| 用户被移出组织 | [organizations.rs#L2099](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L2099) | `send_single_org_removed_from_org` |
| 管理员重置密码 | [organizations.rs#L2943](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L2943) | `send_admin_reset_password` |

### 3.4 紧急访问相关

| 触发事件 | 触发位置 | 邮件函数 |
|----------|----------|----------|
| 紧急访问邀请 | [emergency_access.rs#L265](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L265) | `send_emergency_access_invite` |
| 邀请被接受 | [emergency_access.rs#L377](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L377) | `send_emergency_access_invite_accepted` |
| 邀请被确认 | [emergency_access.rs#L433](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L433) | `send_emergency_access_invite_confirmed` |
| 恢复请求发起 | [emergency_access.rs#L472](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L472) | `send_emergency_access_recovery_initiated` |
| 恢复请求被批准 | [emergency_access.rs#L510](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L510) | `send_emergency_access_recovery_approved` |
| 恢复请求被拒绝 | [emergency_access.rs#L543](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L543) | `send_emergency_access_recovery_rejected` |

### 3.5 定时任务触发

| 任务 | 触发频率 | 邮件函数 | 位置 |
|------|----------|----------|------|
| 2FA 未完成提醒 | 每分钟 | `send_incomplete_2fa_login` | [two_factor/mod.rs#L265](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/mod.rs#L265) |
| 紧急访问恢复超时 | 每小时（7分） | `send_emergency_access_recovery_timed_out` | [emergency_access.rs#L758](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L758) |
| 紧急访问提醒 | 每小时（3分） | `send_emergency_access_recovery_reminder` | [emergency_access.rs#L820](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L820) |

---

## 4. 模板系统详解

### 4.1 模板引擎与结构

使用 **Handlebars** 模板引擎，模板文件分为：
- **纯文本模板** (`.hbs`) - 用于纯文本邮件内容
- **HTML 模板** (`.html.hbs`) - 用于富文本邮件内容

#### 模板分隔格式

每个模板使用 `<!---------------->` 分隔主题和正文：

```handlebars
邮件主题行
<!---------------->
邮件正文内容
{{> email/email_footer_text }}
```

解析逻辑见 [mail.rs#L129-L150](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L129-L150)。

### 4.2 通用模板字段

所有邮件模板都包含以下公共字段：

| 字段 | 说明 | 来源 |
|------|------|------|
| `url` | 站点域名 | `CONFIG.domain()` |
| `img_src` | 图片资源前缀 | `CONFIG._smtp_img_src()` |

### 4.3 各模板字段详解

#### 账户相关

| 模板名称 | 特有字段 | 说明 |
|----------|----------|------|
| `welcome` | 无 | 欢迎邮件 |
| `welcome_must_verify` | `user_id`, `token` | 需验证的欢迎邮件 |
| `verify_email` | `user_id`, `email`, `token` | 邮箱验证邮件 |
| `register_verify_email` | `email` | 注册验证邮件 |
| `delete_account` | `user_id`, `email`, `token` | 账户删除确认 |
| `pw_hint_some` | `hint` | 包含密码提示 |
| `pw_hint_none` | 无 | 无密码提示 |
| `change_email` | `token` | 邮箱变更验证 |
| `change_email_existing` | `existing_address`, `acting_address` | 现有邮箱变更通知 |
| `change_email_invited` | `existing_address`, `acting_address` | 邀请邮箱变更通知 |
| `sso_change_email` | 无 | SSO 邮箱变更提示 |
| `new_device_logged_in` | `ip`, `device_name`, `device_type`, `datetime` | 新设备登录通知 |
| `incomplete_2fa_login` | `ip`, `device_name`, `device_type`, `datetime`, `time_limit` | 2FA 未完成提醒 |
| `twofactor_email` | `token` | 两步验证令牌 |
| `protected_action` | `token` | 受保护操作令牌 |
| `admin_reset_password` | `user_name`, `org_name` | 管理员重置密码 |

#### 组织相关

| 模板名称 | 特有字段 | 说明 |
|----------|----------|------|
| `send_org_invite` | `org_name` | 组织邀请 |
| `invite_accepted` | `email`, `org_name` | 邀请被接受通知 |
| `invite_confirmed` | `org_name` | 邀请被确认通知 |
| `send_2fa_removed_from_org` | `org_name` | 2FA 被组织移除 |
| `send_single_org_removed_from_org` | `org_name` | 用户被移出组织 |

#### 紧急访问相关

| 模板名称 | 特有字段 | 说明 |
|----------|----------|------|
| `send_emergency_access_invite` | `grantor_name` | 紧急访问邀请 |
| `emergency_access_invite_accepted` | `grantee_email` | 邀请被接受 |
| `emergency_access_invite_confirmed` | `grantor_name` | 邀请被确认 |
| `emergency_access_recovery_initiated` | `grantee_name`, `atype`, `wait_time_days` | 恢复请求发起 |
| `emergency_access_recovery_reminder` | `grantee_name`, `atype`, `days_left` | 恢复请求提醒 |
| `emergency_access_recovery_approved` | `grantor_name` | 恢复请求批准 |
| `emergency_access_recovery_rejected` | `grantor_name` | 恢复请求拒绝 |
| `emergency_access_recovery_timed_out` | `grantee_name`, `atype` | 恢复请求超时 |

#### 其他

| 模板名称 | 特有字段 | 说明 |
|----------|----------|------|
| `smtp_test` | 无 | SMTP 测试邮件 |

### 4.4 数据安全清洗 ([mail.rs#L100-L119](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L100-L119))

在渲染模板前，所有用户输入数据都会经过 XSS 清洗：

```rust
fn sanitize_data(data: &mut serde_json::Value) {
    // 使用正则移除所有 HTML 标签，防止 XSS 和 HTML 注入
    static RE: LazyLock<Regex> = LazyLock::new(|| Regex::new(r"<[^>]+>").unwrap());
    // 递归清洗字符串、对象、数组中的所有值
}
```

### 4.5 模板注册与加载 ([config.rs#L1668-L1751](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/config.rs#L1668-L1751))

1. 首先编译时通过 `include_str!` 加载默认模板
2. 然后运行时从 `templates_folder` 目录加载用户自定义模板（可覆盖默认）
3. 支持 `reload_templates` 开发模式，每次请求重新加载模板

---

## 5. 邮件发送流程

### 5.1 完整流程图示

```
调用 send_* 函数
        ↓
1. 生成所需数据（JWT token, URL 参数等）
        ↓
2. 调用 get_text(template_name, data)
        ├─ 数据清洗 sanitize_data
        ├─ 渲染 HTML 模板 get_template(.html)
        └─ 渲染纯文本模板 get_template()
        ↓
3. 调用 send_email(address, subject, html, text)
        ├─ 构建 Message（发件人、收件人、主题、Message-ID）
        ├─ 构建邮件体 MultiPart（可选嵌入图片附件）
        └─ 构建邮件内容 multipart/alternative (纯文本+HTML)
        ↓
4. 调用 send_with_selected_transport(email)
        ├─ 选择传输方式 (SMTP 或 Sendmail)
        └─ 调用对应 transport.send(email)
        ↓
5. 结果处理（成功返回 Ok，失败分类错误）
```

### 5.2 传输层实现

#### SMTP 传输构建 ([mail.rs#L33-L97](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L33-L97))

```rust
fn smtp_transport() -> AsyncSmtpTransport<Tokio1Executor> {
    // 1. 创建基础连接 (host, port, timeout)
    // 2. 配置 TLS (off / starttls / force_tls)
    //    - force_tls: 使用 Wrapper 模式（端口 465）
    //    - starttls: 使用 Required 模式（端口 587）
    //    - 可配置接受无效证书/主机名（危险）
    // 3. 配置认证凭证 (username + password)
    // 4. 配置 HELO 名称
    // 5. 配置认证机制 (Plain, Login, Xoauth2)
}
```

#### Sendmail 传输构建 ([mail.rs#L25-L31](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L25-L31))

```rust
fn sendmail_transport() -> AsyncSendmailTransport<Tokio1Executor> {
    // 使用配置的命令或默认的 sendmail 命令
}
```

#### 传输选择 ([mail.rs#L653-L701](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L653-L701))

根据 `CONFIG.use_sendmail()` 决定使用哪种传输方式。

### 5.3 邮件构建 ([mail.rs#L703-L733](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L703-L733))

#### Message-ID 生成

使用 UUID + 发件人域名确保唯一性：
```rust
.message_id(Some(format!("<{}@{}>", crate::util::get_uuid(), smtp_from.domain())))
```

#### 邮件内容结构

当 `smtp_embed_images = true` 时：
```
multipart/alternative
├─ text/plain (纯文本版本)
└─ multipart/related
   ├─ text/html (HTML 版本)
   ├─ inline attachment: logo-gray.png (image/png)
   └─ inline attachment: mail-github.png (image/png)
```

当 `smtp_embed_images = false` 时：
```
multipart/alternative
├─ text/plain (纯文本版本)
└─ text/html (HTML 版本，图片使用外部 URL)
```

---

## 6. 失败处理机制

### 6.1 错误分类处理 ([mail.rs#L653-L701](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L653-L701))

#### Sendmail 错误

| 错误类型 | 处理方式 |
|----------|----------|
| 客户端错误 | `err!("Sendmail client error: {e}")` |
| 响应错误 | `err!("Sendmail response error: {e}")` |
| 其他错误 | `err!("Sendmail error: {e}")` |

#### SMTP 错误

| 错误类型 | 识别方法 | 处理方式 |
|----------|----------|----------|
| 客户端错误 | `e.is_client()` | `err!("SMTP client error: {e}")` |
| 临时错误 (4xx) | `e.is_transient()` | `err!("SMTP 4xx error: {e}")` |
| 永久错误 (5xx) | `e.is_permanent()` | 特殊处理 535（认证失败），添加额外说明 |
| 超时错误 | `e.is_timeout()` | `err!("SMTP timeout error: {e}")` |
| TLS 错误 | `e.is_tls()` | `err!("SMTP encryption error: {e}")` |
| 其他错误 | - | `err!("SMTP error: {e}")` |

所有错误都会在 debug 级别打印详细信息。

### 6.2 触发点的错误处理策略

#### 宽松策略（仅记录错误，不影响主流程）

大多数场景使用此策略，邮件发送失败不阻止操作继续：

```rust
// 示例: identity.rs#L480-L489
if let Err(e) = mail::send_new_device_logged_in(...).await {
    error!("Error sending new device email: {e:#?}");
    // 如果 require_device_email = false，则继续执行
    if CONFIG.require_device_email() {
        // 只有严格模式下才返回错误
        err!("Could not send login notification email...");
    }
}
```

适用场景：
- 新设备登录通知
- 欢迎邮件
- 组织邀请确认
- 紧急访问非关键通知

#### 严格策略（发送失败则操作失败）

当 `require_device_email = true` 时，新设备登录邮件发送失败会导致登录失败。

#### 必须成功策略（错误直接向上传播）

```rust
// 示例: mail.rs#L559
send_email(address, &subject, body_html, body_text).await
```

使用 `?` 传播错误，调用者必须处理。

适用场景：
- 两步验证令牌发送（`send_token`）
- 受保护操作令牌
- 邮箱验证令牌
- 管理员主动发送的测试邮件

### 6.3 定时任务中的错误处理

在定时任务中，邮件发送失败会被记录但不中断任务：

```rust
// 示例: two_factor/mod.rs#L265-L280
match mail::send_incomplete_2fa_login(...).await {
    Ok(()) => {
        // 发送成功后删除记录
        login.delete(&conn).await
    }
    Err(e) => {
        error!("Error sending incomplete 2FA email: {e:#?}");
        // 不删除记录，下次继续尝试
    }
}
```

---

## 7. 关键调用示例

### 7.1 发送两步验证邮件

```rust
// 触发点: two_factor/email.rs#L109-L123
pub async fn send_token(user_id: &UserId, conn: &DbConn) -> EmptyResult {
    // 1. 查询用户的 2FA 配置
    let mut twofactor = TwoFactor::find_by_user_and_type(...).await?;
    
    // 2. 生成随机令牌
    let generated_token = crypto::generate_email_token(CONFIG.email_token_size());
    
    // 3. 保存令牌到数据库
    let mut twofactor_data = EmailTokenData::from_json(&twofactor.data)?;
    twofactor_data.set_token(generated_token);
    twofactor.data = twofactor_data.to_json();
    twofactor.save(conn).await?;
    
    // 4. 发送邮件
    mail::send_token(&twofactor_data.email, &twofactor_data.last_token?).await?;
    
    Ok(())
}
```

### 7.2 发送新设备登录通知

```rust
// 触发点: identity.rs#L478-L490
if CONFIG.mail_enabled() && device.is_new() {
    let now = Utc::now().naive_utc();
    if let Err(e) = mail::send_new_device_logged_in(
        &user.email, 
        &ip.ip.to_string(), 
        &now, 
        device
    ).await {
        error!("Error sending new device email: {e:#?}");
        if CONFIG.require_device_email() {
            err!("Could not send login notification email...");
        }
    }
}
```

---

## 8. 邮件类型总览

| 邮件函数 | 模板 | 主要用途 |
|----------|------|----------|
| `send_password_hint` | `email/pw_hint_some/none` | 密码提示 |
| `send_delete_account` | `email/delete_account` | 账户删除确认 |
| `send_verify_email` | `email/verify_email` | 邮箱验证 |
| `send_register_verify_email` | `email/register_verify_email` | 注册验证 |
| `send_welcome` | `email/welcome` | 欢迎邮件 |
| `send_welcome_must_verify` | `email/welcome_must_verify` | 需验证的欢迎邮件 |
| `send_2fa_removed_from_org` | `email/send_2fa_removed_from_org` | 2FA 被移除 |
| `send_single_org_removed_from_org` | `email/send_single_org_removed_from_org` | 移出组织 |
| `send_invite` | `email/send_org_invite` | 组织邀请 |
| `send_emergency_access_invite` | `email/send_emergency_access_invite` | 紧急访问邀请 |
| `send_emergency_access_invite_accepted` | `email/emergency_access_invite_accepted` | 邀请被接受 |
| `send_emergency_access_invite_confirmed` | `email/emergency_access_invite_confirmed` | 邀请被确认 |
| `send_emergency_access_recovery_approved` | `email/emergency_access_recovery_approved` | 恢复被批准 |
| `send_emergency_access_recovery_initiated` | `email/emergency_access_recovery_initiated` | 恢复请求发起 |
| `send_emergency_access_recovery_reminder` | `email/emergency_access_recovery_reminder` | 恢复请求提醒 |
| `send_emergency_access_recovery_rejected` | `email/emergency_access_recovery_rejected` | 恢复被拒绝 |
| `send_emergency_access_recovery_timed_out` | `email/emergency_access_recovery_timed_out` | 恢复超时 |
| `send_invite_accepted` | `email/invite_accepted` | 邀请被接受 |
| `send_invite_confirmed` | `email/invite_confirmed` | 邀请被确认 |
| `send_new_device_logged_in` | `email/new_device_logged_in` | 新设备登录 |
| `send_incomplete_2fa_login` | `email/incomplete_2fa_login` | 2FA 未完成提醒 |
| `send_token` | `email/twofactor_email` | 2FA 令牌 |
| `send_change_email` | `email/change_email` | 邮箱变更验证 |
| `send_change_email_existing` | `email/change_email_existing` | 邮箱变更通知 |
| `send_change_email_invited` | `email/change_email_invited` | 邮箱变更通知 |
| `send_sso_change_email` | `email/sso_change_email` | SSO 邮箱变更 |
| `send_test` | `email/smtp_test` | SMTP 测试 |
| `send_admin_reset_password` | `email/admin_reset_password` | 管理员重置密码 |
| `send_protected_action_token` | `email/protected_action` | 受保护操作令牌 |

---

## 9. 安全注意事项

1. **XSS 防护**：所有模板数据在渲染前都会经过 HTML 标签清洗 ([mail.rs#L100-L119](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L100-L119))

2. **敏感信息保护**：配置导出时 SMTP 密码会被掩码为 `***` ([config.rs#L61](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/config.rs#L61))

3. **TLS 配置警告**：`smtp_accept_invalid_certs` 和 `smtp_accept_invalid_hostnames` 有明确的危险警告

4. **认证失败增强**：SMTP 535 错误会额外添加 "Authentication credentials invalid" 提示

5. **严格模式**：`require_device_email` 配置可确保登录通知邮件必须发送成功才能登录
