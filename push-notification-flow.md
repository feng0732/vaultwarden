# Vaultwarden 推送通知集成代码流程分析

## 目录结构与核心文件

| 文件 | 作用 |
|------|------|
| [src/api/push.rs](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs) | 推送通知核心实现 |
| [src/api/notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/notifications.rs) | WebSocket + Push 通知统一入口 |
| [src/db/models/device.rs](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs) | 设备模型与过滤逻辑 |
| [src/config.rs](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/config.rs) | 推送配置项 |

---

## 一、事件触发流程

### 1.1 触发源位置

推送通知的触发点分布在各业务模块中，以 `ciphers.rs` 为例：

- [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/ciphers.rs) 包含 11 处 `send_cipher_update` 调用

### 1.2 典型触发链路

```
API 请求处理
    ↓
业务逻辑完成（保存/更新/删除）
    ↓
调用 nt.send_cipher_update()  [notifications.rs]
    ↓
内部判断 push_enabled
    ↓
调用 push_cipher_update()      [push.rs]
```

### 1.3 支持的推送事件类型 (UpdateType)

定义于 [notifications.rs#L622-L654](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/notifications.rs#L622-L654)：

| 类型值 | 事件 | 触发函数 |
|--------|------|----------|
| 0 | Cipher 更新 | `push_cipher_update` |
| 1 | Cipher 创建 | `push_cipher_update` |
| 2 | Cipher 删除 | `push_cipher_update` |
| 3 | Folder 删除 | `push_folder_update` |
| 5 | Vault 同步 | `push_user_update` |
| 7 | Folder 创建 | `push_folder_update` |
| 8 | Folder 更新 | `push_folder_update` |
| 10 | 设置更新 | `push_user_update` |
| 11 | 用户登出 | `push_logout` |
| 12-14 | Send 操作 | `push_send_update` |
| 15 | 认证请求 | `push_auth_request` |
| 16 | 认证响应 | `push_auth_response` |

---

## 二、设备过滤机制

### 2.1 设备类型过滤

**核心判断**：[device.rs#L95-L97](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L95-L97)

```rust
pub fn is_push_device(&self) -> bool {
    matches!(DeviceType::from_i32(self.atype), DeviceType::Android | DeviceType::Ios)
}
```

仅 **Android (0)** 和 **iOS (1)** 设备支持推送通知。

### 2.2 Push Token 存在性检查

**数据库查询**：[device.rs#L249-L261](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L249-L261)

```rust
pub async fn check_user_has_push_device(user_uuid: &UserId, conn: &DbConn) -> bool {
    conn.run(move |conn| {
        devices::table
            .filter(devices::user_uuid.eq(user_uuid))
            .filter(devices::push_token.is_not_null())
            .count()
            .first::<i64>(conn)
            .ok()
            .unwrap_or(0)
            != 0
    })
    .await
}
```

**关键点**：
- 查询条件：`push_token IS NOT NULL`
- 每个 `push_*` 函数入口都会调用此检查
- 无有效设备时直接返回，不发送请求

### 2.3 设备注册流程

**注册函数**：[push.rs#L87-L136](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L87-L136)

1. 检查 `push_enabled` 配置
2. 检查 `is_push_device()` 设备类型
3. 检查 `push_token` 非空（为空则跳过并警告）
4. 生成唯一 `push_uuid`（如不存在）
5. 调用 Bitwarden Push Relay `/push/register` 接口
6. 保存 `push_uuid` 到数据库

---

## 三、发送入口与数据流转

### 3.1 统一发送入口

**核心发送函数**：[push.rs#L267-L300](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L267-L300)

```rust
async fn send_to_push_relay(notification_data: Value) {
    // 1. 配置检查
    if !CONFIG.push_enabled() { return; }
    
    // 2. 获取 Auth Token
    let auth_api_token = match get_auth_api_token().await { ... };
    
    // 3. 发送 POST 请求到 Push Relay
    //    POST {push_relay_uri}/push/send
    //    Headers: Authorization: Bearer {token}
    //    Body: notification_data (JSON)
}
```

### 3.2 Auth Token 管理

**Token 获取逻辑**：[push.rs#L36-L85](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L36-L85)

- 使用 `LazyLock<RwLock<LocalAuthPushToken>>` 缓存 Token
- Token 有效期设为官方返回值的一半（提前刷新）
- 请求地址：`{push_identity_uri}/connect/token`
- Grant Type: `client_credentials`
- Scope: `api.push`

### 3.3 各类型推送的数据结构

#### Cipher 更新推送
[push.rs#L160-L189](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L160-L189)
```json
{
  "userId": "...",
  "organizationId": null,
  "deviceId": "acting_device_push_uuid",
  "identifier": "acting_device_uuid",
  "type": 0,
  "payload": {
    "id": "cipher_uuid",
    "userId": "...",
    "organizationId": null,
    "collectionIds": null,
    "revisionDate": "ISO8601"
  },
  "clientType": null,
  "installationId": null
}
```

**注意**：组织所属的 Cipher **不发送推送**（第 162-164 行）

#### 用户登出推送
[push.rs#L191-L207](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L191-L207)
- 使用 `tokio::task::spawn` 异步执行，不阻塞请求

### 3.4 异步发送策略

所有推送函数（除 `push_cipher_update`）均使用：
```rust
tokio::task::spawn(send_to_push_relay(...));
```

**目的**：推送通知不阻塞主业务流程的响应。

---

## 四、离线场景兜底机制

### 4.1 双通知通道设计

Vaultwarden 实现了 **WebSocket + Push Relay** 双通道：

**统一调度点**：[notifications.rs#L342-L532](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/notifications.rs#L342-L532)

以 `send_cipher_update` 为例：
```rust
pub async fn send_cipher_update(...) {
    // 跳过条件：两个通道都禁用
    if *NOTIFICATIONS_DISABLED { return; }
    
    // 构建消息数据
    let data = create_update(...);
    
    // 通道 1: WebSocket (在线实时)
    if CONFIG.enable_websocket() {
        self.send_update(&user.uuid, &data).await;
    }
    
    // 通道 2: Push Relay (离线兜底)
    if CONFIG.push_enabled() && user_ids.len() == 1 {
        push_cipher_update(ut, cipher, device, conn).await;
    }
}
```

### 4.2 离线兜底的触发条件

| 场景 | WebSocket 行为 | Push Relay 行为 |
|------|---------------|-----------------|
| 设备在线 | ✓ 实时送达 | ✓ 也发送（冗余） |
| 设备离线 | ✗ 连接断开，消息丢失 | ✓ Relay → FCM/APNs 平台负责离线投递 |
| 应用后台 | ✗ WebSocket 可能被杀死 | ✓ 系统级推送拉起应用 |

> **注意**：Vaultwarden 自身没有离线缓存逻辑，详见第十一章的代码验证。

### 4.3 关键限制

**单用户限制**：`push_cipher_update` 仅在 `user_ids.len() == 1` 时触发  
（[notifications.rs#L451-L453](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/notifications.rs#L451-L453)）

**含义**：
- 个人 Vault 变更 → 发送推送
- 组织共享 Cipher → **不发送推送**（与上游一致）

### 4.4 WebSocket 连接管理

- 连接存储：`WS_USERS: DashMap<UserId, Vec<(entry_uuid, Sender)>>`
- 心跳机制：每 15 秒发送 Ping
- 自动清理：`WSEntryMapGuard` Drop 时从 Map 移除

---

## 五、配置依赖

### 5.1 必需配置项 ([config.rs#L527-L538](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/config.rs#L527-L538))

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `push_enabled` | 总开关 | false |
| `push_relay_uri` | 推送中继地址 | https://push.bitwarden.com |
| `push_identity_uri` | 身份认证地址 | https://identity.bitwarden.com |
| `push_installation_id` | 自托管安装 ID | (空) |
| `push_installation_key` | 自托管安装密钥 | (空) |

### 5.2 配置验证 ([config.rs#L1004-L1032](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/config.rs#L1004-L1032))

- 启用 Push 时，installation_id/key 不能为空
- URI 必须以 `https://` 开头
- URI 格式必须合法

---

## 六、完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    业务事件 (Cipher/Folder/...)             │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  notifications.rs   │
                    │  send_*_update()    │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐
│ NOTIFICATIONS_     │  │ enable_websocket   │  │ push_enabled &&    │
│ DISABLED?          │  │  ?                 │  │ single_user?       │
└─────────┬──────────┘  └─────────┬──────────┘  └─────────┬──────────┘
          │ YES                   │ YES                   │ YES
          ▼                       ▼                       ▼
      [跳过]          ┌────────────────────┐    ┌────────────────────┐
                      │ WS_USERS 查找连接  │    │ check_user_has_    │
                      │  发送二进制消息    │    │ push_device()      │
                      └────────────────────┘    └─────────┬──────────┘
                                                          │
                                                          ▼
                                              ┌────────────────────┐
                                              │ get_auth_api_token │
                                              │ (缓存 + 刷新)       │
                                              └─────────┬──────────┘
                                                          │
                                                          ▼
                                              ┌────────────────────┐
                                              │ POST /push/send    │
                                              │ to Push Relay      │
                                              └────────────────────┘
```

---

## 七、关键代码速查表

| 功能 | 文件位置 |
|------|----------|
| 推送设备类型判断 | [device.rs#L95-L97](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L95-L97) |
| 用户推送设备检查 | [device.rs#L249-L261](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L249-L261) |
| 设备注册流程 | [push.rs#L87-L136](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L87-L136) |
| Auth Token 管理 | [push.rs#L36-L85](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L36-L85) |
| 统一发送函数 | [push.rs#L267-L300](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L267-L300) |
| 双通道路由 | [notifications.rs#L342-L532](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/notifications.rs#L342-L532) |
| WebSocket 连接管理 | [notifications.rs#L27-L80](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/notifications.rs#L27-L80) |

---

## 八、移动端登录 → 设备注册的完整触发路径

### 8.1 两条注册路径

Vaultwarden 中触发 `register_push_device` 有且仅有两条路径：

#### 路径 A：登录成功时自动注册

```
POST /connect/token  (identity.rs login)
    ↓
password_login / sso_login
    ↓
authenticated_response()          ← [identity.rs#L470-L568]
    ↓
if !device.is_new() {             ← 关键条件：仅老设备注册
    register_push_device(device, conn).await?;
}
    ↓
register_push_device()            ← [push.rs#L87-L136]
```

**关键条件 `!device.is_new()`**：
- [identity.rs#L494-L497](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/identity.rs#L494-L497) 显式判断 **仅非新设备** 才注册
- `is_new()` 的判定逻辑：[device.rs#L91-L93](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L91-L93) —— `created_at == updated_at`
- 新建设备在 [identity.rs#L751-L758](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/identity.rs#L751-L758) 中 `save(false, conn)` 保存，`update_time=false` 使得 `created_at == updated_at`，所以 `is_new()` 返回 `true`，跳过注册
- **设计意图**：首次登录的设备还没有 `push_token`（客户端在登录成功后才获取），所以注册无意义

**注意**：`api_key_login`（[identity.rs#L570-L712](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/identity.rs#L570-L712)）**不调用** `register_push_device`，API Key 登录不走推送注册流程。

#### 路径 B：客户端主动上报 Push Token

```
PUT /devices/identifier/<device_id>/token  ← [accounts.rs#L1395-L1420]
    ↓
比对 push_token 是否变化
    ↓ （token 不同时）
device.push_token = Some(token);
device.save(...)
    ↓
register_push_device(&mut device, &conn).await?;
```

**关键逻辑**：[accounts.rs#L1404-L1417](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1404-L1417)

```rust
// 如果 token 与已注册的相同，直接返回，不重复注册
if device.push_token.as_ref() == Some(&token) {
    debug!("Device {device_id} for user {} is already registered and token is identical", ...);
    return Ok(());
}
// token 不同，更新并重新注册
device.push_token = Some(token);
device.save(true, &conn).await?;
register_push_device(&mut device, &conn).await?;
```

**这是移动端最重要的注册路径**：客户端登录成功后，会主动调用此 API 上报 FCM/APNs 的 push_token，触发真正的推送注册。

### 8.2 ConnectData 中的 device_push_token 字段

[identity.rs#L1125-L1127](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/identity.rs#L1125-L1127)：

```rust
#[allow(unused)]
_device_push_token: Option<String>, // Unused; mobile device push not yet supported.
```

**登录请求中客户端发送的 `device_push_token` 字段被标记为未使用**。Push Token 的上报不走登录接口，而是走上面路径 B 的专用 API。

### 8.3 设备注销路径

```
PUT /devices/identifier/<device_id>/clear-token  ← [accounts.rs#L1422-L1441]
    ↓
Device::clear_push_token_by_uuid()    ← 清空数据库中的 push_token
    ↓
unregister_push_device()              ← 调用 Relay /push/delete
```

### 8.4 完整生命周期时序

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   客户端      │     │ Vaultwarden  │     │  Push Relay  │     │   数据库      │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │  1. POST /connect/token  │              │                    │
       │ ──────────────────────→ │              │                    │
       │                         │  新设备:      │                    │
       │                         │  get_device() │                   │
       │                         │  创建 Device  │                   │
       │                         │  (push_uuid=随机, push_token=None)│
       │                         │ ─────────────────────────────────→│
       │  is_new()=true, 跳过注册  │              │                    │
       │                         │              │                    │
       │  2. PUT /devices/.../token │            │                    │
       │  { push_token: "FCM_xxx" } │           │                    │
       │ ──────────────────────→ │              │                    │
       │                         │  token 变更?  │                    │
       │                         │  更新 push_token                  │
       │                         │ ─────────────────────────────────→│
       │                         │  register_push_device()          │
       │                         │  POST /push/register             │
       │                         │ ─────────────→│                   │
       │                         │ ←─────────────│                   │
       │                         │  保存 push_uuid                   │
       │                         │ ─────────────────────────────────→│
       │  200 OK                 │              │                    │
       │ ←────────────────────── │              │                    │
       │                         │              │                    │
       │  3. 后续推送事件         │              │                    │
       │                         │  check_user_has_push_device()    │
       │                         │ ─────────────────────────────────→│
       │                         │ ←─────────────────────────────────│
       │                         │  POST /push/send                  │
       │                         │ ─────────────→│                   │
       │                         │              │  系统推送拉起       │
       │  ← 系统推送通知 ────────────────────────│                   │
       │                         │              │                    │
       │  4. 登出 / 清除 token   │              │                    │
       │  PUT /devices/.../clear-token │        │                    │
       │ ──────────────────────→ │              │                    │
       │                         │  clear_push_token                 │
       │                         │ ─────────────────────────────────→│
       │                         │  POST /push/delete/{push_uuid}   │
       │                         │ ─────────────→│                   │
```

---

## 九、push_uuid / push_token 校验逻辑详解

### 9.1 两个字段的区别

| 字段 | 生成方 | 含义 | 数据库列 |
|------|--------|------|----------|
| `push_uuid` | 服务端生成 | 服务端为每个 user+device 组合分配的唯一标识，用于 Relay 注册/注销/推送中的 `deviceId` | `push_uuid` |
| `push_token` | 客户端上报 | FCM/APNs 的设备推送令牌，用于 Relay 注册时关联推送通道 | `push_token` |

### 9.2 push_uuid 的初始化与补全

**新建设备时**：[device.rs#L52](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L52)

```rust
push_uuid: Some(PushId(get_uuid())),  // 创建时即生成
push_token: None,                      // 等待客户端上报
```

**注册推送设备时**：[push.rs#L100-L103](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L100-L103)

```rust
if device.push_uuid.is_none() {
    device.push_uuid = Some(PushId(get_uuid()));
}
```

双重保障：即使数据库中 `push_uuid` 为 NULL（旧数据迁移等场景），注册时也会补生成。

### 9.3 push_token 的三级校验

#### 校验 1：注册时——push_token 必须存在

[push.rs#L92-L96](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L92-L96)

```rust
if device.push_token.is_none() {
    warn!("Skipping the registration of the device {:?} because the push_token field is empty.", device.uuid);
    warn!("To get rid of this message you need to logout, clear the app data and login again on the device.");
    return Ok(());  // 静默跳过，不报错
}
```

**无 push_token 的设备不会被注册到 Relay**。这是 `register_push_device` 的守门逻辑。

#### 校验 2：发送时——用户至少有一个推送设备

[push.rs#L170](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L170)

```rust
if Device::check_user_has_push_device(user_id, conn).await {
    send_to_push_relay(...).await;
}
```

查询条件 `push_token IS NOT NULL`（[device.rs#L249-L261](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L249-L261)），**仅检查数据库中有无记录，不校验 Relay 端是否注册成功**。

#### 校验 3：Token 上报时——去重

[accounts.rs#L1407-L1409](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1407-L1409)

```rust
if device.push_token.as_ref() == Some(&token) {
    debug!("Device {device_id} for user {} is already registered and token is identical", ...);
    return Ok(());  // token 未变，不重复注册
}
```

### 9.4 push_uuid 为 None 时的推送行为

在 `push_cipher_update`、`push_folder_update` 等函数中，`device.push_uuid` 被直接序列化为 JSON 的 `"deviceId"` 字段。由于 `push_uuid` 类型为 `Option<PushId>`：

- 如果 `push_uuid = None`，JSON 中 `"deviceId": null`
- **代码不做额外校验**——推送仍会发送到 Relay，但 `deviceId` 为 null

**潜在问题**：如果一个设备的 `push_token` 不为空（通过了 `check_user_has_push_device`），但 `push_uuid` 为 NULL，推送会发送但 `deviceId` 为 null。不过正常流程下，`register_push_device` 成功后一定会保存 `push_uuid`，此场景只在 Relay 注册失败时才会出现。

---

## 十、Relay 失败处理：仅记录日志，无重试

### 10.1 `send_to_push_relay` 的错误处理

[push.rs#L267-L300](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L267-L300)

```rust
async fn send_to_push_relay(notification_data: Value) {
    // 守门：配置未启用则直接返回
    if !CONFIG.push_enabled() { return; }

    // 错误 1：获取 Auth Token 失败
    let auth_api_token = match get_auth_api_token().await {
        Ok(s) => s,
        Err(e) => {
            debug!("Could not get the auth push token: {e}");  // ← 仅 debug 日志
            return;                                             // ← 直接返回
        }
    };

    // 错误 2：构造 HTTP 请求失败
    let req = match make_http_request(Method::POST, &(CONFIG.push_relay_uri() + "/push/send")) {
        Ok(r) => r,
        Err(e) => {
            error!("An error occurred while sending a send update to the push relay: {e}");  // ← 仅 error 日志
            return;                                                                          // ← 直接返回
        }
    };

    // 错误 3：HTTP 请求发送失败
    if let Err(e) = req
        .header(...)
        .json(&notification_data)
        .send()
        .await
    {
        error!("An error occurred while sending a send update to the push relay: {e}");  // ← 仅 error 日志
        // 无 return，函数自然结束
    }
}
```

**结论**：三种失败场景均 **只记录日志，不重试，不回补**。

### 10.2 `register_push_device` 的错误处理

[push.rs#L119-L129](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L119-L129)

```rust
if let Err(e) = make_http_request(Method::POST, &(...))?
    .header(...)
    .json(&data)
    .send()
    .await?
    .error_for_status()
{
    err!(format!("An error occurred while proceeding registration of a device: {e}"));
    // ← 返回错误给调用者！
}
```

**注意**：注册失败会通过 `err!` 宏向上传播错误，导致 API 返回 500。
- 路径 A（登录时注册）：`authenticated_response` 中 `register_push_device(device, conn).await?;` —— 注册失败会导致**登录失败**
- 路径 B（上报 Token 时）：同样 `await?` —— 上报 Token 的 API 会返回错误

**但**：由于 `is_push_device()` 和 `push_token.is_none()` 的前置检查，非推送设备会静默跳过，不会触发 Relay 请求。

### 10.3 `get_auth_api_token` 的错误处理

[push.rs#L62-L74](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L62-L74)

```rust
// 请求 Identity 服务失败
let res = match make_http_request(...).form(&params).send().await {
    Ok(r) => r,
    Err(e) => err!(format!("Error getting push token from bitwarden server: {e}")),
};

// 解析 Token 响应失败
let json_pushtoken = match res.json::<AuthPushToken>().await {
    Ok(r) => r,
    Err(e) => err!(format!("Unexpected push token received from bitwarden server: {e}")),
};
```

**Auth Token 获取失败通过 `err!` 传播**，但在 `send_to_push_relay` 中被 `match` 捕获后仅打 debug 日志。**整个链条中没有任何重试逻辑**。

### 10.4 错误处理差异总结

| 场景 | 函数 | 错误传播方式 | 是否影响业务 API | 是否重试 |
|------|------|-------------|-----------------|---------|
| Auth Token 获取失败 | `get_auth_api_token` | `err!` 向上传播 | 在 `send_to_push_relay` 中被吞掉 | 否 |
| 注册设备失败 | `register_push_device` | `err!` 向上传播 | **是**，登录/Token上报返回500 | 否 |
| 注销设备失败 | `unregister_push_device` | `err!` 向上传播 | 取决于调用方 | 否 |
| 推送发送失败 | `send_to_push_relay` | 仅打日志 | **否**，静默丢弃 | 否 |

---

## 十一、关于"离线缓存"的代码验证

### 11.1 之前文档中的说法

> "设备离线 → Push Relay 负责离线缓存"

### 11.2 代码验证结论：Vaultwarden 自身 **没有** 离线缓存逻辑

遍历推送相关代码：

1. **`send_to_push_relay`**：发送后即丢弃，无论成功失败都不存储
2. **`WebSocketUsers::send_update`**：通过 `mpsc::channel` 发送，如果接收端已断开，`sender.send()` 会返回错误，仅打日志
3. **没有任何队列、重试、持久化机制**

### 11.3 "离线缓存"的实际含义

Push Relay（Bitwarden 官方托管服务）收到推送请求后，会通过 FCM/APNs 向移动端发送系统级推送。FCM/APNs 各自有消息缓存机制（如 FCM 的 collapsible messages、APNs 的 QoS），**这是由推送平台提供的，不是 Vaultwarden 代码实现的**。

更准确的说法是：

| 机制 | 实现方 | 代码位置 |
|------|--------|---------|
| 推送消息发送 | Vaultwarden | `send_to_push_relay` |
| 离线消息缓存 | FCM/APNs 推送平台 | 无 Vaultwarden 代码 |
| 设备上线后主动同步 | Bitwarden 客户端 | 客户端逻辑，非服务端 |
| WebSocket 离线补偿 | **无** | 连接断开期间消息直接丢弃 |

### 11.4 修正后的离线场景描述

```
场景：设备 A 离线时，设备 B 修改了 Cipher

1. Vaultwarden 处理修改请求
2. notifications.rs 同时走两个通道:
   a. WebSocket: 尝试向设备 A 的 WS 连接发消息
      → 连接已断开，sender.send() 失败，打 error 日志
      → 消息丢失，无重放
   b. Push Relay: 调用 send_to_push_relay()
      → 请求发送到 Bitwarden Push Relay
      → Push Relay 通过 FCM/APNs 推送系统通知
      → FCM/APNs 在设备上线后投递（平台行为）
3. 设备 A 上线后:
   a. 收到 FCM/APNs 的系统推送 → 拉起应用
   b. 客户端主动调用 /sync 获取最新数据
   c. WebSocket 重连 → 后续变更可实时接收
```

**关键认知**：推送通知本身不携带完整的 Vault 数据，仅是一个"触发信号"。客户端收到推送后，通过 `/sync` 主动拉取最新数据。因此即使推送丢失，用户手动刷新也能恢复一致——只是时效性下降。
