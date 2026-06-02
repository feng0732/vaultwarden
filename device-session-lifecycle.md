# 设备与会话生命周期分析

## 1. 核心数据模型

### 1.1 设备模型 (Device)

**定义位置**: [src/db/models/device.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs#L18-L36)

```rust
pub struct Device {
    pub uuid: DeviceId,              // 设备唯一标识
    pub user_uuid: UserId,            // 关联用户
    pub name: String,                 // 设备名称
    pub atype: i32,                   // 设备类型枚举
    pub refresh_token: String,        // 刷新令牌（存储在数据库）
    pub twofactor_remember: Option<String>,  // 2FA 记住令牌
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,
}
```

**主键设计**: 复合主键 `(uuid, user_uuid)`，允许同一个设备 UUID 在不同用户间复用。

**刷新令牌生成**: [generate_refresh_token()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs#L59-L62)
- 64 字节随机数，Base64URL 编码
- 每个设备拥有独立的刷新令牌

### 1.2 用户安全戳 (Security Stamp)

**定义位置**: [src/db/models/user.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L56-L57)

```rust
pub struct User {
    pub security_stamp: String,       // 安全戳，UUID 格式
    pub stamp_exception: Option<String>,  // 安全戳例外（JSON）
}
```

**安全戳例外结构**: [UserStampException](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L101-L105)
```rust
pub struct UserStampException {
    pub routes: Vec<String>,          // 例外路由列表
    pub security_stamp: String,       // 旧安全戳
    pub expire: i64,                  // 过期时间（2分钟）
}
```

---

## 2. 登录流程与令牌生成

### 2.1 登录入口

**位置**: [src/api/identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/identity.rs#L59-L136)

支持的登录方式：
- `password` - 密码登录
- `refresh_token` - 刷新令牌登录
- `client_credentials` - API Key 登录
- `authorization_code` - SSO 登录

### 2.2 设备获取或创建

**位置**: [get_device()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/identity.rs#L743-L759)

```rust
async fn get_device(data: &ConnectData, conn: &DbConn, user: &User) -> ApiResult<Device> {
    // 1. 解析设备类型（数字或字符串）
    let device_type = util::try_parse_string(data.device_type.as_ref()).unwrap_or(14);
    
    // 2. 查找现有设备
    if let Some(device) = Device::find_by_uuid_and_user(&device_id, &user.uuid, conn).await {
        return Ok(device);
    }
    
    // 3. 创建设备（自动生成刷新令牌）
    let mut device = Device::new(device_id, user.uuid.clone(), device_name, device_type);
    device.save(false, conn).await?;
    Ok(device)
}
```

**设备新建判定**: [is_new()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs#L91-L93)
- `created_at == updated_at` 表示新设备
- 新设备登录会发送邮件通知

### 2.3 双因素认证

**位置**: [twofactor_auth()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/identity.rs#L761-L793)

- 检查用户是否启用 2FA
- 如果启用，标记登录为不完整状态
- 验证通过后生成 `twofactor_token`

### 2.4 令牌生成

**位置**: [AuthTokens::new()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L1221-L1245)

生成两种令牌：

#### 访问令牌 (Access Token)
- **类型**: JWT (RS256 签名)
- **有效期**: 默认 2 小时 ([DEFAULT_ACCESS_VALIDITY](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L41))
- **包含字段**:
  - `sub` - 用户 UUID
  - `sstamp` - 用户安全戳
  - `device` - 设备 ID
  - `devicetype` - 设备类型字符串
  - `client_id` - 客户端类型
  - `scope` - 权限范围

#### 刷新令牌 (Refresh Token)
- **类型**: JWT (RS256 签名)
- **有效期**: 
  - 桌面/浏览器: 30 天 ([DEFAULT_REFRESH_VALIDITY](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L39))
  - 移动端: 90 天 ([MOBILE_REFRESH_VALIDITY](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L40))
- **包含字段**:
  - `sub` - 认证方式 (Password/Sso/UserApiKey)
  - `device_token` - 数据库中存储的刷新令牌（用于校验）

**认证响应**: [authenticated_response()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/identity.rs#L470-L568)
```json
{
    "access_token": "<JWT>",
    "refresh_token": "<JWT>",
    "expires_in": 7200,
    "token_type": "Bearer",
    "scope": "api offline_access"
}
```

---

## 3. 令牌验证机制

### 3.1 请求守卫验证

**位置**: [Headers::from_request()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L611-L690)

每个 API 请求经过以下验证步骤：

1. **提取访问令牌**: 从 `Authorization: Bearer <token>` 头部提取
2. **JWT 解码验证**: [decode_login()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L129-L131)
   - 验证签名
   - 验证过期时间 (exp)
   - 验证生效时间 (nbf)
   - 验证签发者 (iss)
3. **设备存在性检查**: 根据 `device` 和 `sub` 查找设备
4. **用户存在性检查**: 根据 `sub` 查找用户
5. **安全戳验证** (核心机制):

```rust
if user.security_stamp != claims.sstamp {
    // 检查是否有安全戳例外
    if let Some(stamp_exception) = user.stamp_exception {
        // 验证例外未过期、路由匹配、安全戳匹配
    } else {
        err_handler!("Invalid security stamp");
    }
}
```

### 3.2 刷新令牌验证

**位置**: [refresh_tokens()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L1248-L1299)

```rust
pub async fn refresh_tokens(
    ip: &ClientIp,
    refresh_token: &str,
    client_id: Option<String>,
    conn: &DbConn,
) -> ApiResult<(Device, AuthTokens)> {
    // 1. 解码刷新令牌 JWT
    let refresh_claims = decode_refresh(refresh_token)?;
    
    // 2. 根据 device_token 查找设备（数据库中的刷新令牌）
    let Some(mut device) = Device::find_by_refresh_token(&refresh_claims.device_token, conn).await else {
        err!("Invalid refresh token");
    };
    
    // 3. 更新设备 updated_at
    device.save(true, conn).await?;
    
    // 4. 生成新的访问令牌和刷新令牌
    let auth_tokens = AuthTokens::new(&device, &user, refresh_claims.sub, client_id);
    
    Ok((device, auth_tokens))
}
```

**关键**: 刷新令牌 JWT 中的 `device_token` 必须与数据库中存储的 `refresh_token` 完全一致。

---

## 4. 会话强制失效机制

### 4.1 管理员踢出用户

**位置**: [deauth_user()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/admin.rs#L463-L482)

```rust
async fn deauth_user(user_id: UserId, ...) -> EmptyResult {
    // 1. 发送登出通知（WebSocket + Push）
    nt.send_logout(&user, None, &conn).await;
    
    // 2. 注销推送设备
    for device in Device::find_push_devices_by_user(&user.uuid, &conn).await {
        unregister_push_device(device.push_uuid.as_ref()).await;
    }
    
    // 3. 删除所有设备记录
    Device::delete_all_by_user(&user.uuid, &conn).await?;
    
    // 4. 重置安全戳 + 轮换所有刷新令牌
    user.reset_security_stamp(&conn).await?;
    
    user.save(&conn).await
}
```

### 4.2 禁用用户

**位置**: [disable_user()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/admin.rs#L484-L497)

```rust
async fn disable_user(user_id: UserId, ...) -> EmptyResult {
    // 1. 重置安全戳 + 轮换刷新令牌
    user.reset_security_stamp(&conn).await?;
    
    // 2. 标记用户为禁用
    user.enabled = false;
    user.save(&conn).await?;
    
    // 3. 发送登出通知
    nt.send_logout(&user, None, &conn).await;
    
    // 4. 删除所有设备
    Device::delete_all_by_user(&user.uuid, &conn).await?;
}
```

### 4.3 安全戳重置

**位置**: [reset_security_stamp()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L217-L221)

```rust
pub async fn reset_security_stamp(&mut self, conn: &DbConn) -> EmptyResult {
    // 1. 生成新的安全戳（UUID）
    self.security_stamp = get_uuid();
    
    // 2. 轮换用户所有设备的刷新令牌
    Device::rotate_refresh_tokens_by_user(&self.uuid, conn).await?;
    
    Ok(())
}
```

### 4.4 刷新令牌轮换

**位置**: [rotate_refresh_tokens_by_user()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs#L263-L272)

```rust
pub async fn rotate_refresh_tokens_by_user(user_uuid: &UserId, conn: &DbConn) -> EmptyResult {
    // 为每个设备生成独立的新刷新令牌
    let devices = Self::find_by_user(user_uuid, conn).await;
    for mut device in devices {
        device.refresh_token = Device::generate_refresh_token();
        device.save(false, conn).await?;
    }
    Ok(())
}
```

**注意**: 无法使用单条 SQL UPDATE，因为每个设备需要唯一的随机令牌。

---

## 5. 密码变更后的处理

### 5.1 密码变更流程

**位置**: [post_password()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L513-L550)

```rust
async fn post_password(data: Json<ChangePassData>, headers: Headers, ...) -> EmptyResult {
    // 1. 验证旧密码
    if !user.check_valid_password(&data.master_password_hash) {
        err!("Invalid password");
    }
    
    // 2. 记录密码变更事件
    log_user_event(EventType::UserChangedPassword as i32, ...);
    
    // 3. 设置新密码 + 重置安全戳
    user.set_password(
        &data.new_master_password_hash,
        Some(data.key),
        true,  // reset_security_stamp = true
        Some(vec![  // 安全戳例外路由
            "post_rotatekey",
            "get_contacts",
            "get_public_keys",
            "get_api_webauthn",
        ]),
        &conn,
    ).await?;
    
    // 4. 保存用户
    let save_result = user.save(&conn).await;
    
    // 5. 发送登出通知（排除当前设备）
    nt.send_logout(&user, Some(&headers.device), &conn).await;
    
    save_result
}
```

### 5.2 设置密码的内部逻辑

**位置**: [set_password()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L192-L215)

```rust
pub async fn set_password(
    &mut self,
    password: &str,
    new_key: Option<String>,
    reset_security_stamp: bool,
    allow_next_route: Option<Vec<String>>,
    conn: &DbConn,
) -> EmptyResult {
    // 1. 哈希新密码
    self.password_hash = crypto::hash_password(...);
    
    // 2. 设置安全戳例外（如果提供）
    if let Some(route) = allow_next_route {
        self.set_stamp_exception(route);
    }
    
    // 3. 更新用户密钥
    if let Some(new_key) = new_key {
        self.akey = new_key;
    }
    
    // 4. 重置安全戳（如果需要）
    if reset_security_stamp {
        self.reset_security_stamp(conn).await?;
    }
    
    Ok(())
}
```

### 5.3 安全戳例外机制

**位置**: [set_stamp_exception()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L230-L237)

```rust
pub fn set_stamp_exception(&mut self, route_exception: Vec<String>) {
    let stamp_exception = UserStampException {
        routes: route_exception,
        security_stamp: self.security_stamp.clone(),  // 旧安全戳
        expire: (Utc::now() + TimeDelta::try_minutes(2).unwrap()).timestamp(),
    };
    self.stamp_exception = Some(serde_json::to_string(&stamp_exception).unwrap());
}
```

**验证例外**: [auth.rs#L654-L680](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L654-L680)

当安全戳不匹配时：
1. 检查是否存在 `stamp_exception`
2. 检查例外是否过期（2分钟）
3. 检查当前路由是否在例外列表中
4. 检查例外中的安全戳是否与 JWT 中的一致

**目的**: 允许密码变更后的后续请求（如密钥轮换）在短时间内继续使用旧令牌完成操作。

### 5.4 KDF 变更的处理

**位置**: [post_kdf()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L615-L648)

与密码变更类似，但：
- 不需要例外路由（`allow_next_route: None`）
- 同样排除当前设备发送登出通知

### 5.5 邮箱变更的处理

**位置**: [post_email()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L1005-L1055)

- 需要重新哈希密码（因为邮箱是 KDF 的盐）
- 重置安全戳，不设例外
- **不排除当前设备**，所有设备全部登出

---

## 6. 主动登出通知机制

### 6.1 登出通知发送

**位置**: [send_logout()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/notifications.rs#L362-L381)

```rust
pub async fn send_logout(&self, user: &User, acting_device: Option<&Device>, conn: &DbConn) {
    let acting_device_id = acting_device.map(|d| d.uuid.clone());
    
    // 构造登出消息
    let data = create_update(
        vec![
            ("UserId".into(), user.uuid.to_string().into()),
            ("Date".into(), serialize_date(user.updated_at)),
        ],
        UpdateType::LogOut,
        acting_device_id,  // 排除的设备 ID
    );
    
    // 通过 WebSocket 发送
    if CONFIG.enable_websocket() {
        self.send_update(&user.uuid, &data).await;
    }
    
    // 通过 Push 发送
    if CONFIG.push_enabled() {
        push_logout(user, acting_device, conn).await;
    }
}
```

### 6.2 客户端处理

客户端收到 `LogOut` 消息后：
- 如果消息中的 `acting_device_id` 与自身设备 ID 相同，则忽略
- 否则，清除本地凭据，强制用户重新登录

---

## 7. 完整生命周期流程图

```
用户登录
    ↓
[identity.rs] get_device() → 查找或创建设备
    ↓
[identity.rs] twofactor_auth() → 2FA 验证（如需要）
    ↓
[auth.rs] AuthTokens::new() → 生成 access_token + refresh_token
    ↓
[identity.rs] authenticated_response() → 返回令牌给客户端
    ↓
正常使用
    ↓
客户端请求 API
    ↓
[auth.rs] Headers::from_request()
    ├─ 解码 JWT access_token
    ├─ 验证签名、过期、签发者
    ├─ 检查设备存在性
    └─ 验证 security_stamp 匹配
        ├─ 匹配 → 继续处理
        └─ 不匹配 → 检查 stamp_exception
            ├─ 有效例外 → 继续处理
            └─ 无效 → 返回 401 错误
    ↓
密码变更 / 管理员踢出
    ↓
[user.rs] reset_security_stamp()
    ├─ 生成新 security_stamp
    └─ [device.rs] rotate_refresh_tokens_by_user()
        └─ 为每个设备生成新 refresh_token
    ↓
[notifications.rs] send_logout()
    ├─ WebSocket 广播登出消息
    └─ Push 推送登出消息
    ↓
其他客户端收到登出消息 → 清除凭据 → 强制重新登录
    ↓
后续请求使用旧令牌
    ↓
security_stamp 不匹配 + 无有效例外 → 返回 401
    ↓
refresh_token 轮换，旧 refresh_token 失效
    ↓
用户需要重新登录
```

---

## 8. 关键设计要点

### 8.1 双重失效机制

1. **安全戳失效**: 使所有现存的 access_token 立即失效
2. **刷新令牌轮换**: 使所有现存的 refresh_token 立即失效

双重保险确保即使 access_token 被泄露，也无法通过 refresh_token 获取新令牌。

### 8.2 当前设备豁免

密码变更和 KDF 变更时，通过 `acting_device` 参数排除当前设备：
- 避免客户端在操作完成后立即被登出
- 提升用户体验
- 邮箱变更等敏感操作不豁免

### 8.3 安全戳例外

允许特定路由在 2 分钟内使用旧安全戳：
- 支持密钥轮换等需要多步操作的场景
- 过期自动失效
- 严格限制路由范围

### 8.4 设备类型区分

- 移动端 refresh_token 有效期更长（90天 vs 30天）
- 不同设备类型有不同的推送策略
- CLI 设备有特殊处理逻辑

---

## 9. 管理员强制失效：删设备与令牌轮换的执行顺序差异

三条管理员级别的强制失效路径虽然都同时触发了「删除设备」和「重置安全戳」，但**执行顺序不同**，导致令牌轮换的实际效果截然不同。

### 9.1 三条路径的执行顺序对比

| 步骤 | [deauth_user](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/admin.rs#L463-L482) | [disable_user](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/admin.rs#L484-L497) | [post_sstamp](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L919-L934) |
|------|-----------|-------------|------------|
| 1 | `send_logout` | `reset_security_stamp` | `reset_security_stamp` |
| 2 | 注销推送设备 | 设 `enabled=false` | `user.save` |
| 3 | **`delete_all_by_user`** | `user.save` | `send_logout` |
| 4 | **`reset_security_stamp`** | `send_logout` | **`delete_all_by_user`** |
| 5 | `user.save` | **`delete_all_by_user`** | — |

### 9.2 关键差异：令牌轮换是空操作还是有意义操作

#### deauth_user：先删设备，再轮换 → 轮换是空操作

```rust
// admin.rs L478-L479
Device::delete_all_by_user(&user.uuid, &conn).await?;   // ← 先删，设备表已清空
user.reset_security_stamp(&conn).await?;                  // ← 再轮换，但已无设备可轮换
```

[reset_security_stamp()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L217-L221) 内部调用 [rotate_refresh_tokens_by_user()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs#L263-L272)，后者执行 `Self::find_by_user()` 查找该用户的所有设备。此时设备记录已经被删光，返回空 Vec，for 循环体不执行。

**实际效果**：
- 安全戳已更新 → 旧 access_token 立即失效 ✓
- 设备记录已删除 → [find_by_refresh_token()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs#L222-L225) 查不到任何设备 → 旧 refresh_token 也失效 ✓
- 令牌轮换代码跑了一遍，但对空列表操作 → **无副作用，属于冗余调用**

#### disable_user：先轮换，再删设备 → 轮换执行了但结果被覆盖

```rust
// admin.rs L487
user.reset_security_stamp(&conn).await?;   // ← 先轮换，设备还在，逐个生成新 refresh_token
// ...中间保存用户、发送通知...
// admin.rs L494
Device::delete_all_by_user(&user.uuid, &conn).await?;  // ← 再删，刚轮换的设备全删
```

**实际效果**：
- 轮换确实执行了（每个设备的 `refresh_token` 字段被更新写入数据库）
- 但紧接着设备记录被全删，刚写入的新令牌随之消失
- 最终效果与 deauth_user 一样：安全戳变了、设备没了，旧令牌全部失效
- 区别仅是**多做了一次无用的数据库写入**

#### post_sstamp：先轮换，再删设备 → 与 disable_user 同理

```rust
// accounts.rs L926
user.reset_security_stamp(&conn).await?;   // ← 先轮换
// accounts.rs L929
nt.send_logout(&user, None, &conn).await;  // ← 通知所有设备（无豁免）
// accounts.rs L931
Device::delete_all_by_user(&user.uuid, &conn).await?;  // ← 再删
```

与 disable_user 执行顺序相同，轮换的写入同样被后续删除覆盖。

### 9.3 为什么三条路径的最终效果相同

尽管执行顺序不同，三条路径的最终效果**完全一致**：

| 维度 | 最终状态 |
|------|---------|
| `user.security_stamp` | 已更新为新 UUID |
| 设备记录 | 全部删除 |
| 旧 access_token | 安全戳不匹配 → 失效 |
| 旧 refresh_token | 设备记录不存在 → `find_by_refresh_token` 返回 None → 失效 |
| WebSocket/Push 通知 | 所有设备收到登出消息 |

**结论**：当同时做了「删设备」和「轮换令牌」两件事时，删设备本身就足以让旧 refresh_token 失效（因为找不到匹配的设备记录），令牌轮换只是额外的安全冗余。对于 `deauth_user` 这种先删再轮换的路径，轮换更是纯空操作。

---

## 10. 密码变更与 KDF 调整后当前设备的令牌可用性

密码变更和 KDF 变更都通过 [set_password()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L192-L215) 触发安全戳重置和刷新令牌轮换，但传入参数不同，导致当前设备持有的两种令牌可用性存在关键差异。

### 10.1 set_password 内部的执行时序

```rust
// user.rs L192-L215
pub async fn set_password(&mut self, ..., allow_next_route: Option<Vec<String>>, ...) -> EmptyResult {
    self.password_hash = crypto::hash_password(...);

    if let Some(route) = allow_next_route {
        self.set_stamp_exception(route);       // ① 先保存旧安全戳到例外（如果提供）
    }

    if let Some(new_key) = new_key {
        self.akey = new_key;
    }

    if reset_security_stamp {
        self.reset_security_stamp(conn).await?; // ② 再生成新安全戳 + 轮换刷新令牌
    }
    Ok(())
}
```

时序关键点：**stamp_exception 在 reset_security_stamp 之前设置**，因此例外中保存的是变更前的旧安全戳，恰好与当前设备 access_token 中嵌入的 `sstamp` 一致。

### 10.2 两种场景下令牌状态对比

#### 密码变更 (post_password)

[post_password()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L513-L550) 调用：
```rust
user.set_password(
    &data.new_master_password_hash,
    Some(data.key),
    true,                          // reset_security_stamp = true
    Some(vec![                     // allow_next_route = 有例外路由
        "post_rotatekey",
        "get_contacts",
        "get_public_keys",
        "get_api_webauthn",
    ]),
    &conn,
).await?;
nt.send_logout(&user, Some(&headers.device), &conn).await;  // 排除当前设备
```

| 令牌类型 | 变更后状态 | 原因 |
|---------|-----------|------|
| **access_token** | ⚠️ 部分可用（2分钟宽限） | sstamp 不匹配，但有 stamp_exception；仅 `post_rotatekey`/`get_contacts`/`get_public_keys`/`get_api_webauthn` 四个路由可通过验证 |
| **refresh_token** | ❌ 不可用 | reset_security_stamp 内部调用 rotate_refresh_tokens_by_user，数据库中当前设备的 refresh_token 已被替换；JWT 中的 device_token 查不到匹配记录 |

#### KDF 变更 (post_kdf)

[post_kdf()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L615-L648) 调用：
```rust
user.set_password(
    &data.authentication_data.master_password_authentication_hash,
    Some(data.unlock_data.master_key_wrapped_user_key),
    true,    // reset_security_stamp = true
    None,    // allow_next_route = 无例外路由！
    &conn,
).await?;
nt.send_logout(&user, Some(&headers.device), &conn).await;  // 排除当前设备
```

| 令牌类型 | 变更后状态 | 原因 |
|---------|-----------|------|
| **access_token** | ❌ 立即不可用 | sstamp 不匹配，且**无 stamp_exception**；任何路由的请求都会返回 401 |
| **refresh_token** | ❌ 不可用 | 同密码变更，数据库中的 refresh_token 已被轮换 |

### 10.3 令牌可用性汇总表

| 场景 | access_token | refresh_token | stamp_exception | 设备豁免通知 |
|------|-------------|---------------|-----------------|------------|
| 密码变更 | ⚠️ 4个路由可用2分钟 | ❌ 不可用 | ✅ 有（4路由/2分钟） | ✅ 当前设备豁免 |
| KDF 变更 | ❌ 立即不可用 | ❌ 不可用 | ❌ 无 | ✅ 当前设备豁免 |
| 邮箱变更 | ❌ 立即不可用 | ❌ 不可用 | ❌ 无 | ❌ 全部设备通知 |
| 密钥轮换 | ❌ 立即不可用 | ❌ 不可用 | ❌ 无 | ✅ 当前设备豁免 |
| 管理员踢出 | ❌ 立即不可用 | ❌ 设备已删 | ❌ 无 | ❌ 全部设备通知 |
| 禁用用户 | ❌ 立即不可用 | ❌ 设备已删 | ❌ 无 | ❌ 全部设备通知 |
| 安全戳重置 | ❌ 立即不可用 | ❌ 设备已删 | ❌ 无 | ❌ 全部设备通知 |

### 10.4 为什么密码变更需要 stamp_exception 而 KDF 变更不需要

密码变更的客户端流程是多步操作：

```
1. POST /accounts/password          ← 密码变更本身（此时 stamp_exception 生效）
2. POST /accounts/key-management/rotate-user-account-keys  ← 重新加密所有数据
3. 客户端提示用户重新登录
```

密码变更后，客户端必须紧接着调用密钥轮换接口来重新加密所有数据。这两步之间，客户端只能依靠 stamp_exception 继续通过身份验证。

而 KDF 变更是原子操作——客户端在一次请求中同时提交新的认证数据和加密密钥，无需后续步骤，因此不需要例外窗口。

### 10.5 stamp_exception 在密钥轮换后的状态

[post_rotatekey()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs#L797-L917) 内部再次调用 `set_password(..., true, None, &conn)`，即**再次重置安全戳但不设置新的 stamp_exception**。

注意 [reset_security_stamp()](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs#L217-L221) 不会清除 `stamp_exception` 字段。但此时 `stamp_exception` 中保存的旧安全戳与新生成的安全戳已完全不同，且例外的2分钟窗口在正常流程下也已过期，因此**残留的 stamp_exception 不会被误用**。

过期清理机制：当请求守卫检测到 stamp_exception 已过期时，[auth.rs L664-L671](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs#L664-L671) 会主动调用 `user.reset_stamp_exception()` 并保存用户记录，将过期的例外从数据库中清除。

---

## 11. 相关文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 设备模型 | [src/db/models/device.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/device.rs) |
| 用户模型 | [src/db/models/user.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/db/models/user.rs) |
| 认证系统 | [src/auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/auth.rs) |
| 登录流程 | [src/api/identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/identity.rs) |
| 账户管理 | [src/api/core/accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/core/accounts.rs) |
| 管理员功能 | [src/api/admin.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/admin.rs) |
| 通知系统 | [src/api/notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/3-vaultwarden/src/api/notifications.rs) |
