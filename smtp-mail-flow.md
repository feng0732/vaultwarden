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

---

## 10. 关键邮件流程深度解析

### 10.1 邀请码邮件流程（send_invite）

#### 触发入口
- **组织邀请**：[organizations.rs#L1113](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1113)
- **管理端邀请**：[admin.rs#L307-L335](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L307-L335)
- **重发邀请**：[organizations.rs#L1208-L1262](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1208-L1262)

#### 完整处理流程

```
调用 send_invite(user, org_id, member_id, org_name, invited_by_email)
        ↓
1. 生成 JWT Claims ([auth.rs#L310-L329](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/auth.rs#L310-L329))
   ├─ nbf: 当前时间戳（Not Before）
   ├─ exp: 当前时间 + invitation_expiration_hours（默认5小时）
   ├─ iss: "invite" (JWT_INVITE_ISSUER)
   ├─ sub: user_id
   ├─ email: 用户邮箱
   ├─ org_id: 组织ID
   ├─ member_id: 成员ID
   └─ invited_by_email: 邀请人邮箱（可选）
        ↓
2. JWT 编码 → invite_token
        ↓
3. 构建 URL 查询参数 ([mail.rs#L297-L313](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L297-L313))
   ├─ email
   ├─ organizationName
   ├─ organizationId
   ├─ organizationUserId
   ├─ token (JWT invite_token)
   ├─ orgSsoIdentifier (仅当 sso_enabled && sso_only 时，值为 org_id)
   └─ orgUserHasExistingUser (仅当 user.private_key.is_some() 时，值为 "true")
        ↓
4. 构建最终 URL 格式：
   {domain}/#/accept-organization/?{query_string}
        ↓
5. 模板渲染 ([send_org_invite.hbs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email/send_org_invite.hbs))
   ├─ 传入字段：url, img_src, org_name
   └─ 模板仅显示：org_name（组织名称）和完整的 url 链接
        ↓
6. 发送邮件
```

#### 🔍 URL 参数 vs 模板字段对比

| 数据项 | 存在于 URL 参数 | 存在于模板字段 | 说明 |
|--------|----------------|----------------|------|
| `email` | ✅ | ❌ | 仅在URL中，用于客户端识别 |
| `organizationName` | ✅ | ❌ | 仅在URL中，用于客户端显示 |
| `organizationId` | ✅ | ❌ | 仅在URL中，用于客户端API调用 |
| `organizationUserId` | ✅ | ❌ | 仅在URL中，用于客户端API调用 |
| `token` (JWT) | ✅ | ❌ | 仅在URL中，用于验证邀请有效性 |
| `orgSsoIdentifier` | ✅ (条件) | ❌ | SSO 专用参数 |
| `orgUserHasExistingUser` | ✅ (条件) | ❌ | 标识用户是否已有账户 |
| `org_name` | ❌ | ✅ | 仅在邮件正文中显示组织名称 |
| `url` (完整链接) | ❌ | ✅ | 邮件中完整可点击链接 |
| `img_src` | ❌ | ✅ | HTML 模板图片前缀 |

#### 🔑 JWT Claims 结构 ([InviteJwtClaims](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/auth.rs#L287-L308))

```rust
pub struct InviteJwtClaims {
    pub nbf: i64,              // Not Before - 令牌生效时间
    pub exp: i64,              // Expiration - 令牌过期时间
    pub iss: String,           // Issuer - 签发者，固定为 "invite"
    pub sub: UserId,           // Subject - 被邀请用户ID
    pub email: String,         // 被邀请用户邮箱
    pub org_id: OrganizationId, // 目标组织ID
    pub member_id: MembershipId, // 成员关系ID
    pub invited_by_email: Option<String>, // 邀请人邮箱
}
```

#### 错误处理
- **必须成功策略**：使用 `?` 传播错误，调用者必须处理
- 组织邀请流程中发送失败会导致邀请操作整体失败
- 管理端邀请失败返回 500 Internal Server Error

---

### 10.2 验证链接邮件流程

#### 10.2.1 邮箱验证邮件（send_verify_email）

##### 触发入口
- 用户主动请求验证：[accounts.rs#L1065](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1065)
- 登录时自动发送（未验证邮箱）：[identity.rs#L445](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L445)

##### 完整处理流程

```
调用 send_verify_email(address, user_id)
        ↓
1. 生成 JWT Claims ([auth.rs#L501-L510](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/auth.rs#L501-L510))
   ├─ nbf: 当前时间戳
   ├─ exp: 当前时间 + invitation_expiration_hours
   ├─ iss: "verify_email" (JWT_VERIFYEMAIL_ISSUER)
   └─ sub: user_id.to_string()
        ↓
2. JWT 编码 → verify_email_token
        ↓
3. 模板字段构建 ([mail.rs#L193-L202](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L193-L202))
   ├─ url: CONFIG.domain()
   ├─ img_src
   ├─ user_id
   ├─ email: percent_encode(address, NON_ALPHANUMERIC)
   └─ token: verify_email_token
        ↓
4. 模板内拼接完整 URL ([verify_email.hbs#L5](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email/verify_email.hbs#L5))
   {{url}}/#/verify-email/?userId={{user_id}}&token={{token}}
        ↓
5. 模板渲染
   ├─ 显示的完整链接包含：url + /#/verify-email/?userId={{user_id}}&token={{token}}
   └─ email 字段仅用于 JWT，不显示在邮件正文中
        ↓
6. 发送邮件
```

##### 🔍 字段流向说明

| 字段 | 传入模板 | 模板内显示 | URL 拼接位置 |
|------|----------|------------|-------------|
| `user_id` | ✅ | ✅ (在URL中) | `?userId={{user_id}}` |
| `token` (JWT) | ✅ | ✅ (在URL中) | `&token={{token}}` |
| `email` (编码后) | ✅ | ❌ | 仅传入模板，不用于URL拼接 |
| `url` | ✅ | ✅ | 链接前缀 `{{url}}/#/verify-email/` |

##### JWT Claims 结构 ([BasicJwtClaims](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/auth.rs#L485-L499))

```rust
pub struct BasicJwtClaims {
    pub nbf: i64,              // Not Before
    pub exp: i64,              // Expiration
    pub iss: String,           // Issuer - "verify_email"
    pub sub: String,           // Subject - user_id
}
```

##### 错误处理

| 触发位置 | 错误策略 | 代码位置 |
|----------|----------|----------|
| 主动请求验证 | 必须成功，使用 `?` | [accounts.rs#L1065](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1065) |
| 登录时自动发送 | 宽松策略，仅记录错误，不阻止登录 | [identity.rs#L445-L447](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L445-L447) |

---

#### 10.2.2 注册验证邮件（send_register_verify_email）

##### 触发入口
- 注册流程中：[identity.rs#L1072](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L1072)

##### 完整处理流程

```
调用 send_register_verify_email(email, token)
        ↓
1. 构建 URL 查询参数 ([mail.rs#L208-L212](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L208-L212))
   ├─ email
   └─ token (注册令牌，由上游生成)
        ↓
2. 构建完整 URL
   {domain}/#/finish-signup/?email={email}&token={token}
        ↓
3. 模板渲染 ([register_verify_email.hbs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email/register_verify_email.hbs))
   ├─ 传入字段：url (完整URL), img_src, email
   └─ 模板显示：完整的 url 链接
        ↓
4. 发送邮件
```

##### 🔍 字段流向说明

| 字段 | 存在于 URL 参数 | 传入模板 | 模板显示 |
|------|----------------|----------|----------|
| `email` | ✅ | ✅ | ❌ (仅传入，不单独显示) |
| `token` | ✅ | ❌ | ❌ (仅在URL中) |
| `url` (完整链接) | ❌ | ✅ | ✅ (作为可点击链接) |

##### 错误处理
- **必须成功策略**：使用 `?` 传播错误，注册流程失败
- 上游 token 生成逻辑：[identity.rs#L1037-L1056](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L1037-L1056)

---

#### 10.2.3 欢迎邮件（需验证，send_welcome_must_verify）

##### 触发入口
- 用户注册（需验证）：[accounts.rs#L320](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L320)

##### URL 拼接方式（模板内）

```handlebars
{{url}}/#/verify-email/?userId={{user_id}}&token={{token}}
```

模板：[welcome_must_verify.hbs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email/welcome_must_verify.hbs)

---

#### 10.2.4 删除账户邮件（send_delete_account）

##### 触发入口
- 用户请求删除账户：[accounts.rs#L1115](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1115)

##### URL 拼接方式（模板内）

```handlebars
{{url}}/#/verify-recover-delete?userId={{user_id}}&token={{token}}&email={{email}}
```

模板：[delete_account.hbs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email/delete_account.hbs)

---

### 10.3 紧急访问邮件流程

#### 10.3.1 紧急访问邀请（send_emergency_access_invite）

##### 触发入口
- 发起紧急访问邀请：[emergency_access.rs#L265](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L265)
- 重发紧急访问邀请：[emergency_access.rs#L281-L323](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L281-L323)

##### 完整处理流程

```
调用 send_emergency_access_invite(address, user_id, emer_id, grantor_name, grantor_email)
        ↓
1. 生成 JWT Claims ([auth.rs#L348-L367](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/auth.rs#L348-L367))
   ├─ nbf: 当前时间戳
   ├─ exp: 当前时间 + invitation_expiration_hours
   ├─ iss: "emergency_access_invite" (JWT_EMERGENCY_ACCESS_INVITE_ISSUER)
   ├─ sub: user_id (grantor 用户ID)
   ├─ email: grantee 邮箱
   ├─ emer_id: 紧急访问ID
   ├─ grantor_name: 授权人姓名
   └─ grantor_email: 授权人邮箱
        ↓
2. JWT 编码 → token
        ↓
3. 构建 URL 查询参数 ([mail.rs#L348-L356](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L348-L356))
   ├─ id: emer_id.to_string()
   ├─ name: grantor_name
   ├─ email: address (grantee 邮箱)
   └─ token: JWT token
        ↓
4. 构建完整 URL
   {domain}/#/accept-emergency/?{query_string}
        ↓
5. 模板渲染 ([send_emergency_access_invite.hbs](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/static/templates/email/send_emergency_access_invite.hbs))
   ├─ 传入字段：url (完整URL), img_src, grantor_name
   └─ 模板显示：grantor_name 和完整的 url 链接
        ↓
6. 发送邮件
```

##### 🔍 URL 参数 vs 模板字段对比

| 数据项 | 存在于 URL 参数 | 传入模板 | 模板显示 | 说明 |
|--------|----------------|----------|----------|------|
| `id` (emer_id) | ✅ | ❌ | ❌ | 用于客户端API调用 |
| `name` (grantor_name) | ✅ | ❌ | ❌ | URL 中用于客户端显示 |
| `email` (grantee) | ✅ | ❌ | ❌ | URL 中用于客户端识别 |
| `token` (JWT) | ✅ | ❌ | ❌ | 用于验证邀请有效性 |
| `grantor_name` | ❌ | ✅ | ✅ | 邮件正文中显示授权人姓名 |
| `url` (完整链接) | ❌ | ✅ | ✅ | 邮件中完整可点击链接 |
| `img_src` | ❌ | ✅ | ✅ | HTML 模板图片前缀 |

##### JWT Claims 结构 ([EmergencyAccessInviteJwtClaims](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/auth.rs#L331-L346))

```rust
pub struct EmergencyAccessInviteJwtClaims {
    pub nbf: i64,              // Not Before
    pub exp: i64,              // Expiration
    pub iss: String,           // Issuer - "emergency_access_invite"
    pub sub: UserId,           // Subject - grantor user_id
    pub email: String,         // grantee email
    pub emer_id: EmergencyAccessId,
    pub grantor_name: String,
    pub grantor_email: String,
}
```

##### 错误处理
- **必须成功策略**：使用 `?` 传播错误

---

#### 10.3.2 紧急访问恢复超时（定时任务）

##### 触发方式
- 定时任务：每小时第 7 分钟执行
- 入口函数：[emergency_access.rs#L727-L775](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L727-L775)

##### 处理逻辑
```
查询所有 RecoveryInitiated 状态的紧急访问记录
        ↓
检查 recovery_initiated_at + wait_time_days <= now
        ↓
更新状态为 RecoveryApproved
        ↓
发送邮件 (使用 .expect("Error on sending email"))
├─ send_emergency_access_recovery_timed_out → grantor
└─ send_emergency_access_recovery_approved → grantee
```

##### 错误处理
- **强制崩溃策略**：使用 `.expect()`，发送失败会导致程序 panic
- 理由：这是关键安全操作，必须确保通知送达

---

#### 10.3.3 紧急访问恢复提醒（定时任务）

##### 触发方式
- 定时任务：每小时第 3 分钟执行
- 入口函数：[emergency_access.rs#L777-L834](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L777-L834)

##### 处理逻辑
```
查询所有 RecoveryInitiated 状态的紧急访问记录
        ↓
检查：
├─ final_recovery_reminder_at <= now (wait_time_days - 1 天前发起)
└─ next_recovery_reminder_at <= now (每天最多提醒一次)
        ↓
更新 last_notification_at = now
        ↓
发送邮件 send_emergency_access_recovery_reminder → grantor
   (使用 .expect("Error on sending email"))
```

##### 错误处理
- **强制崩溃策略**：使用 `.expect()`，发送失败会导致程序 panic

---

## 11. 管理端触发功能入口

### 11.1 SMTP 测试邮件

#### API 入口
```
POST /admin/test/smtp
```
代码位置：[admin.rs#L337-L346](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L337-L346)

#### 请求体
```json
{
  "email": "test@example.com"
}
```

#### 处理流程
1. 检查 `CONFIG.mail_enabled()` 是否为 true
2. 调用 `mail::send_test(&data.email).await`
3. 发送使用 `smtp_test` 模板
4. **错误处理**：必须成功策略，使用 `?` 传播错误

---

### 11.2 管理端邀请用户

#### API 入口
```
POST /admin/invite
```
代码位置：[admin.rs#L307-L335](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L307-L335)

#### 请求体
```json
{
  "email": "user@example.com"
}
```

#### 处理逻辑
- 检查用户是否已存在（409 Conflict）
- 创建新用户（无密码）
- 发送邀请邮件（使用虚拟 org_id: FAKE_ADMIN_UUID 或 FAKE_SSO_IDENTIFIER）
- **错误处理**：必须成功，失败返回 500 Internal Server Error

---

### 11.3 管理端重发邀请

#### API 入口
```
POST /admin/users/{user_id}/invite/resend
```
代码位置：[admin.rs#L516-L538](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L516-L538)

#### 前置检查
- 用户必须存在（404 NotFound）
- 用户必须尚未接受邀请（password_hash 为空）（400 BadRequest）

#### 错误处理
- **必须成功策略**：使用 `?` 传播错误

---

### 11.4 组织内重发邀请

#### 单个重发
```
POST /organizations/{org_id}/users/{member_id}/reinvite
```
代码位置：[organizations.rs#L1208-L1219](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1208-L1219)

#### 批量重发
```
POST /organizations/{org_id}/users/reinvite
```
代码位置：[organizations.rs#L1173-L1206](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1173-L1206)

#### 前置检查 ([reinvite_member_impl](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1221-L1262))
- 成员必须存在且属于该组织
- 成员状态必须是 `Invited` (0)，已接受/已确认无法重发

#### 错误处理
- 单个重发：必须成功策略
- 批量重发：宽松策略，每个成员独立处理，错误记录在响应中不中断整体流程
  ```rust
  let err_msg = match reinvite_member_impl(...).await {
      Ok(()) => String::new(),
      Err(e) => format!("{e:?}"),
  };
  ```

---

### 11.5 紧急访问重发邀请

#### API 入口
```
POST /emergency-access/{emer_id}/reinvite
```
代码位置：[emergency_access.rs#L281-L323](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L281-L323)

#### 前置检查
- 紧急访问记录必须存在且属于当前用户
- 状态必须是 `Invited` (0)

---

## 12. 定时任务触发入口总览

| 任务名称 | 触发频率 | 调度时间 | 入口函数 |
|----------|----------|----------|----------|
| 2FA 未完成提醒 | 每分钟 | 每分钟第 0 秒 | [send_incomplete_2fa_notifications](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/mod.rs#L244-L284) |
| 紧急访问恢复超时 | 每小时 | 每小时第 7 分钟 | [emergency_request_timeout_job](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L727-L775) |
| 紧急访问恢复提醒 | 每小时 | 每小时第 3 分钟 | [emergency_notification_reminder_job](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L777-L834) |

### 定时任务调度配置
代码位置：[main.rs#L358-L395](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/main.rs#L358-L395)

---

## 13. 错误处理策略完整矩阵

### 13.1 策略类型定义

| 策略类型 | 代码模式 | 行为说明 |
|----------|----------|----------|
| **必须成功** | `.await?` | 错误直接向上传播，调用者必须处理，通常导致操作失败 |
| **严格条件** | `if let Err(e) = ... { if CONFIG.require_*() { err!() } }` | 配置项控制是否失败，默认宽松 |
| **宽松捕获** | `if let Err(e) = ... { error!("..."); }` | 仅记录错误日志，不影响主流程继续执行 |
| **批量宽松** | 循环内 `match` 每个操作 | 每个元素独立处理，错误不中断整体批量操作 |
| **重试保留** | `match { Ok(()) => delete_record, Err(e) => error!(e) }` | 失败不删除数据库记录，下次定时任务继续尝试 |
| **强制崩溃** | `.await.expect("...")` | 发送失败直接 panic，终止程序 |

### 13.2 各场景错误处理策略对照表

| 邮件类型 | 触发场景 | 处理策略 | 数据库记录 | 代码位置 |
|----------|----------|----------|------------|----------|
| **send_invite** | 组织邀请 | 必须成功 | 成员状态保持 Invited | [organizations.rs#L1113](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1113) |
| **send_invite** | 管理端邀请 | 必须成功 | 用户已创建但无法登录 | [admin.rs#L331](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L331) |
| **send_invite** | 组织批量重发 | 批量宽松 | 每个成员独立 | [organizations.rs#L1187-L1190](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1187-L1190) |
| **send_invite** | 管理端重发 | 必须成功 | - | [admin.rs#L531](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L531) |
| **send_verify_email** | 主动请求验证 | 必须成功 | - | [accounts.rs#L1065](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1065) |
| **send_verify_email** | 登录时自动发送 | 宽松捕获 | 登录仍可继续 | [identity.rs#L445-L447](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L445-L447) |
| **send_register_verify_email** | 注册流程 | 必须成功 | 注册失败 | [identity.rs#L1072](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L1072) |
| **send_welcome_must_verify** | 注册（需验证） | 必须成功 | - | [accounts.rs#L320](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L320) |
| **send_welcome** | 注册（无需验证） | 宽松捕获 | 登录正常 | [accounts.rs#L324](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L324) |
| **send_delete_account** | 删除账户请求 | 必须成功 | - | [accounts.rs#L1115](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1115) |
| **send_password_hint** | 密码提示 | 必须成功 | - | [accounts.rs#L1212](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L1212) |
| **send_new_device_logged_in** | 新设备登录 | 严格条件<br>(require_device_email) | 默认宽松<br>严格时登录失败 | [identity.rs#L480-L489](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L480-L489) |
| **send_token** (2FA) | 登录时发送 | 必须成功 | 登录失败 | [identity.rs#L982](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L982) |
| **send_token** (2FA) | 配置 2FA 验证 | 必须成功 | - | [email.rs#L188](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/email.rs#L188) |
| **send_incomplete_2fa_login** | 定时任务 | 重试保留 | 发送成功才删除记录 | [two_factor/mod.rs#L265-L282](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/mod.rs#L265-L282) |
| **send_protected_action_token** | 受保护操作 | 必须成功 | - | [protected_actions.rs#L94](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/protected_actions.rs#L94) |
| **send_emergency_access_invite** | 发起邀请 | 必须成功 | - | [emergency_access.rs#L265](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L265) |
| **send_emergency_access_invite** | 重发邀请 | 必须成功 | - | [emergency_access.rs#L306](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L306) |
| **send_emergency_access_invite_accepted** | 邀请被接受 | 宽松捕获 | - | [emergency_access.rs#L377](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L377) |
| **send_emergency_access_recovery_initiated** | 发起恢复 | 宽松捕获 | - | [emergency_access.rs#L472](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L472) |
| **send_emergency_access_recovery_approved** | 批准恢复 | 宽松捕获 | - | [emergency_access.rs#L510](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L510) |
| **send_emergency_access_recovery_rejected** | 拒绝恢复 | 宽松捕获 | - | [emergency_access.rs#L543](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L543) |
| **send_emergency_access_recovery_timed_out** | 定时任务-超时 | 强制崩溃 | 状态已更新 | [emergency_access.rs#L758](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L758) |
| **send_emergency_access_recovery_reminder** | 定时任务-提醒 | 强制崩溃 | 通知日期已更新 | [emergency_access.rs#L820](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/emergency_access.rs#L820) |
| **send_invite_accepted** | 邀请被接受 | 宽松捕获 | - | [core/mod.rs#L296](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/mod.rs#L296) |
| **send_invite_confirmed** | 邀请被确认 | 宽松捕获 | - | [organizations.rs#L1320](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L1320) |
| **send_2fa_removed_from_org** | 组织移除2FA | 必须成功 | - | [two_factor/mod.rs#L186](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/two_factor/mod.rs#L186) |
| **send_single_org_removed_from_org** | 移出组织 | 宽松捕获 | - | [organizations.rs#L2099](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L2099) |
| **send_change_email\*** | 邮箱变更 | 必须成功 | - | [accounts.rs#L962-L982](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/accounts.rs#L962-L982) |
| **send_sso_change_email** | SSO邮箱变更 | 必须成功 | - | [identity.rs#L330](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/identity.rs#L330) |
| **send_test** | SMTP测试 | 必须成功 | - | [admin.rs#L342](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/admin.rs#L342) |
| **send_admin_reset_password** | 管理员重置密码 | 必须成功 | - | [organizations.rs#L2943](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/api/core/organizations.rs#L2943) |

### 13.3 策略选择逻辑总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    何时使用哪种策略？                            │
├─────────────────────────────────────────────────────────────────┤
│ 🔴 必须成功 (?)                                                 │
│   ├─ 验证类邮件（邮箱验证、注册验证、删除确认）                   │
│   ├─ 2FA 令牌邮件（用户需要令牌才能继续操作）                     │
│   ├─ 邀请类邮件（创建邀请后必须送达）                            │
│   ├─ 管理员主动触发的操作（测试邮件、重置密码）                   │
│   └─ 安全相关变更（邮箱变更、2FA 移除）                          │
├─────────────────────────────────────────────────────────────────┤
│ 🟡 严格条件 (if CONFIG.require_*)                               │
│   └─ 新设备登录通知（默认宽松，可配置为严格）                     │
├─────────────────────────────────────────────────────────────────┤
│ 🟢 宽松捕获 (if let Err(e) = ... { error! })                    │
│   ├─ 通知类邮件（欢迎邮件、邀请接受/确认通知）                   │
│   ├─ 登录后自动发送的验证邮件（不阻止登录）                       │
│   ├─ 组织操作通知（移出组织、紧急访问非关键通知）                 │
│   └─ 批量操作中的非关键通知                                      │
├─────────────────────────────────────────────────────────────────┤
│ 🔵 重试保留 (match Ok→delete, Err→keep)                         │
│   └─ 定时任务发送的 2FA 未完成提醒（失败下次重试）                │
├─────────────────────────────────────────────────────────────────┤
│ ⚫ 强制崩溃 (.expect())                                         │
│   └─ 定时任务中的紧急访问超时和提醒（关键安全操作，必须送达）      │
└─────────────────────────────────────────────────────────────────┘
```

### 13.4 错误分类与日志级别

| 错误类型 | 日志级别 | 说明 |
|----------|----------|------|
| SMTP 客户端错误 | `debug!` + `err!` | 详细错误在 debug，用户消息在 error |
| SMTP 4xx 临时错误 | `debug!` + `err!` | 可重试错误 |
| SMTP 5xx 永久错误 | `debug!` + `err!` | 535 认证失败有额外提示 |
| SMTP 超时 | `debug!` + `err!` | 网络或服务器响应慢 |
| SMTP TLS 错误 | `debug!` + `err!` | 加密连接问题 |
| Sendmail 错误 | `debug!` + `err!` | 命令执行问题 |
| 模板渲染错误 | `err!` | 模板语法或数据问题 |
| 上游调用方捕获 | `error!` | 触发点使用 if let Err 记录 |

代码位置：[mail.rs#L653-L701](file:///d:/fz/0601/solo-dogfeeding/code/10-vaultwarden/src/mail.rs#L653-L701)
