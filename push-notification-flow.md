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

---

## 十二、设备定位：Token 上报与清理的双轨逻辑

### 12.1 两个 API 的定位机制对比

Vaultwarden 提供了两个与 Push Token 管理相关的 API，它们的设备定位逻辑**完全不同**：

| API 端点 | 设备定位方式 | 用户校验 | 路径参数 `device_id` 的作用 |
|---------|-------------|---------|---------------------------|
| `PUT /devices/identifier/<device_id>/token` | 认证头 `headers.device.uuid` | ✓ JWT 认证 | **被忽略** |
| `PUT /devices/identifier/<device_id>/clear-token` | 路径参数 `device_id` | ✗ 无认证 | **作为唯一依据** |

---

### 12.2 Token 上报：认证头优先，路径参数弃用

**函数定义**：[accounts.rs#L1395-L1420](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1395-L1420)

```rust
async fn put_device_token(
    device_id: DeviceId,        // ← 路径参数，存在但不使用！
    data: Json<PushToken>,
    headers: Headers,           // ← 认证头，包含 device 和 user
    conn: DbConn
) -> EmptyResult {
    let token = data.push_token;

    // 🔴 关键：查询使用的是 headers.device.uuid，不是路径参数 device_id
    let Some(mut device) = Device::find_by_uuid_and_user(
        &headers.device.uuid,    // ← 来自 JWT token 的 device
        &headers.user.uuid,      // ← 来自 JWT token 的 user
        &conn
    ).await else {
        // 错误消息却显示路径参数的 device_id，造成误导
        err!(format!("Error: device {device_id} should be present before a token can be assigned"))
    };

    // token 去重检查
    if device.push_token.as_ref() == Some(&token) {
        debug!("Device {device_id} for user {} is already registered and token is identical", ...);
        return Ok(());
    }

    // 更新 token 并注册
    device.push_token = Some(token);
    device.save(true, &conn).await?;
    register_push_device(&mut device, &conn).await?;

    Ok(())
}
```

#### 定位机制分析

1. **认证头设备信息的来源**：
   - `Headers` 是 Rocket 的 `FromRequest` 守卫，会从请求的 `Authorization: Bearer <JWT>` 中解析 token
   - JWT 中包含 `device` claim，对应用户登录时的 `device_id`
   - `Headers::from_request` 通过 `Device::find_by_uuid_and_user(&device_id, &user_id, &conn)` 查询设备
   - 因此 `headers.device` 一定是**当前登录用户的有效设备**

2. **路径参数的命运**：
   - 路径 `device_id` 仅出现在错误消息和 debug 日志中
   - 实际查询完全不使用路径参数
   - **即使路径参数与认证头设备不一致，也不会报错**，静默使用认证头的设备

3. **安全边界**：
   - ✓ JWT 认证保证用户身份
   - ✓ 设备必须属于当前用户
   - ✗ 不校验路径参数与认证设备的一致性

#### 实际影响

客户端必须确保以下三点一致，否则会出现"操作了错误的设备"的问题：
- 登录请求的 `device_identifier`
- JWT token 中携带的 `device` claim
- 上报 Token API 路径中的 `device_id`

由于 Vaultwarden 静默使用认证头设备，即使路径参数写错，也只会静默给"当前登录设备"设置 token，不会报错。

---

### 12.3 Token 清理：路径参数优先，无认证

**函数定义**：[accounts.rs#L1422-L1441](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1422-L1441)

```rust
async fn put_clear_device_token(
    device_id: DeviceId,    // ← 唯一的设备定位依据
    conn: DbConn
    // 🔴 关键：没有 headers: Headers 参数！
) -> EmptyResult {
    if !CONFIG.push_enabled() {
        return Ok(());
    }

    // 🔴 关键：直接用 device_id 查询，不校验用户归属
    if let Some(device) = Device::find_by_uuid(&device_id, &conn).await {
        // 清空数据库中的 push_token
        Device::clear_push_token_by_uuid(&device_id, &conn).await?;
        // 向 Relay 发送注销请求
        unregister_push_device(device.push_uuid.as_ref()).await?;
    }

    Ok(())
}
```

#### 定位机制分析

1. **没有认证守卫**：
   - 函数参数中没有 `headers: Headers`
   - Rocket 不会触发 `Headers::from_request` 的 JWT 认证
   - **理论上任何人都可以调用此 API**

2. **设备定位逻辑**：
   - 直接使用 `Device::find_by_uuid(&device_id, &conn)` 查询
   - `find_by_uuid` 只按 `device.uuid` 主键查询，**不校验用户归属**
   - 只要 device_id 存在，就会清空其 push_token 并注销

3. **代码注释的自我说明**：
   ```rust
   // This is somehow not implemented in any app, added it in case it is required
   // 2025: Also, it looks like it only clears the first found device upstream, which is probably faulty.
   //       This because currently multiple accounts could be on the same device/app and that would cause issues.
   //       Vaultwarden removes the push-token for all devices, but this probably means we should also unregister all these devices.
   ```
   - 此 API **未被任何客户端实际使用**，仅为预留接口
   - 上游 Bitwarden 也有类似问题（只清空第一个找到的设备）
   - 注释本身就承认设计可能有缺陷

4. **实际安全状态**：
   - 虽然没有认证，但由于：
     - `device_id` 是 UUID（熵足够高，难以暴力枚举）
     - 仅能清空 push_token，无法访问或修改其他数据
     - 影响有限（用户再次上报 token 即可恢复）
   - 风险较低但设计不一致

---

### 12.4 两个 API 的定位逻辑差异汇总

| 对比维度 | 设置 Token API | 清理 Token API |
|---------|---------------|---------------|
| 认证机制 | `Headers` JWT 认证 | 无认证 |
| 设备定位依据 | `headers.device.uuid` | 路径参数 `device_id` |
| 路径参数作用 | 仅用于日志/错误消息 | 唯一查询条件 |
| 用户归属校验 | ✓ 设备必须属于当前用户 | ✗ 不校验 |
| 实际使用状态 | ✓ 移动端登录后主动调用 | ✗ 无客户端使用（预留接口） |
| 代码位置 | [accounts.rs#L1395-L1420](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1395-L1420) | [accounts.rs#L1422-L1441](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1422-L1441) |

---

### 12.5 对 push_uuid / push_token 校验边界的影响

这种双轨定位设计导致了校验边界的不一致：

#### 场景 A：设置 Token（认证头优先）

```
调用 PUT /devices/identifier/DEVICE_A/token
    ↓
Headers JWT 解析出设备为 DEVICE_B
    ↓
查询并设置 DEVICE_B 的 token ← 路径参数 DEVICE_A 被忽略
    ↓
register_push_device 使用 DEVICE_B 的 push_uuid 和 push_token
```

**校验边界**：在 `Headers::from_request` 阶段就确保了设备合法性，后续操作都在安全范围内。

#### 场景 B：清理 Token（路径参数优先）

```
调用 PUT /devices/identifier/DEVICE_X/clear-token
    ↓
无 Headers，无认证
    ↓
直接查询 DEVICE_X
    ↓
清空 DEVICE_X 的 token 并注销其 push_uuid
```

**校验边界**：仅依赖 UUID 的不可预测性，没有业务层面的校验。

#### 一致性问题

1. **push_uuid 的一致性风险**：
   - 设置 Token：push_uuid 由认证头设备提供，经过 JWT 校验
   - 清理 Token：push_uuid 直接从查询到的设备读取，无校验

2. **多用户共享设备场景**：
   - 同一物理设备（同一 device.app）上登录多个账号时，会有多条 Device 记录共享相同的 `device.uuid`
   - 设置 Token：只会给"当前登录用户的那条设备记录"设置
   - 清理 Token：`find_by_uuid` 可能找到多个用户的设备，但实际只处理第一条（SQL 排序不确定）

3. **潜在的错位**：
   - 用户 A 登录设备 D → JWT 设备为 D_A（user=A, device=D）
   - 用户 B 登录设备 D → JWT 设备为 D_B（user=B, device=D）
   - 设备 D 的 app 调用清理 API 传入 `device_id=D`
   - 会清空哪个用户的 token？取决于 `find_by_uuid` 的查询结果顺序

---

### 12.6 结论：为什么会有这种差异？

从代码和注释可以推断出设计演进的历史原因：

1. **设置 Token API** 是核心功能，需要严格认证，使用 JWT 中的设备信息确保安全
2. **清理 Token API** 是预留/边缘功能，可能是：
   - 为了方便设备端在无 JWT 上下文时调用（如 app 卸载/登出时）
   - 或者想设计为全局设备标识的清理（一个 app 清理所有用户）
   - 但最终没有被任何客户端实际使用，成为了半吊子实现

**实际使用建议**：如果需要启用/使用清理 Token 的功能，应该：
- 补充 `Headers` 认证守卫
- 统一按用户 + 设备定位
- 或者明确设计为"按 device.app 全局清理所有关联用户"的语义

---

## 十三、清理 Push Token 时多账号受影响的深度分析

### 13.1 DeviceId 的格式与复合主键

#### 数据库主键定义

[schema.rs#L47-L59](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/schema.rs#L47-L59)：

```sql
devices (uuid, user_uuid) {
    uuid         -> Text,        -- 客户端生成的设备标识
    user_uuid    -> Text,        -- 所属用户
    push_uuid    -> Nullable<Text>,
    push_token   -> Nullable<Text>,
    ...
}
```

**复合主键为 `(uuid, user_uuid)`**，这意味着：

- 同一个 `uuid`（设备标识）可以出现在**多行**中，只要 `user_uuid` 不同
- 每行代表"某个用户在某台设备上的会话"

#### DeviceId 的语义

[device.rs#L369-L372](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L369-L372)：

```rust
pub struct DeviceId(String);
```

`DeviceId` 是一个包裹 `String` 的新类型，**不包含 `user_uuid`**。它仅代表设备端生成的标识符（由客户端在登录时通过 `device_identifier` 字段提供）。

#### 多账号场景的数据布局

当同一台手机（device_id=`abc123`）上登录了两个用户时：

| uuid (DeviceId) | user_uuid | push_uuid | push_token |
|-----------------|-----------|-----------|------------|
| `abc123` | `user_A` | `push-uuid-A` | `FCM_token_A` |
| `abc123` | `user_B` | `push-uuid-B` | `FCM_token_B` |

两条记录共享相同的 `uuid`，但各自有独立的 `push_uuid` 和 `push_token`。

---

### 13.2 `find_by_uuid` 与 `clear_push_token_by_uuid` 的查询差异

这是多账号受影响的**核心根源**：两个函数对复合主键的使用方式完全不同。

#### `find_by_uuid`：仅按 uuid 过滤，返回单条记录

[device.rs#L208-L210](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L208-L210)

```rust
pub async fn find_by_uuid(uuid: &DeviceId, conn: &DbConn) -> Option<Self> {
    conn.run(move |conn| {
        devices::table
            .filter(devices::uuid.eq(uuid))    // ← 仅过滤 uuid
            .first::<Self>(conn)                // ← 只取第一条
            .ok()
    })
    .await
}
```

**关键问题**：
- `filter(devices::uuid.eq(uuid))` 只用 `uuid` 过滤，**不加** `user_uuid` 条件
- 当存在多条记录（多用户共享设备）时，`.first()` 返回哪一条**取决于数据库的默认排序**
- SQLite/MySQL/PostgreSQL 的默认排序可能不一致
- **调用者拿到的是"随机"某个用户的设备记录**

#### `clear_push_token_by_uuid`：仅按 uuid 过滤，批量更新

[device.rs#L212-L221](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/db/models/device.rs#L212-L221)

```rust
pub async fn clear_push_token_by_uuid(uuid: &DeviceId, conn: &DbConn) -> EmptyResult {
    conn.run(move |conn| {
        diesel::update(devices::table)
            .filter(devices::uuid.eq(uuid))     // ← 仅过滤 uuid
            .set(devices::push_token.eq::<Option<String>>(None))
            .execute(conn)                      // ← 影响所有匹配行
            .map_res("Error removing push token")
    })
    .await
}
```

**关键差异**：
- `UPDATE ... WHERE uuid = ?` **不加** `user_uuid` 条件
- `.execute()` 是批量操作，**会清空所有匹配行的 push_token**
- 这意味着**所有在该设备上登录的用户的 push_token 都会被清空**

#### 两个函数的行为对比

| 函数 | WHERE 条件 | 结果范围 | 多账号影响 |
|------|----------|---------|-----------|
| `find_by_uuid` | `uuid = ?` | 返回 1 条（随机） | 只拿到"某个"用户的设备 |
| `clear_push_token_by_uuid` | `uuid = ?` | 更新所有匹配行 | **所有用户**的 token 被清空 |
| `find_by_uuid_and_user` | `uuid = ? AND user_uuid = ?` | 精确到用户 | 仅影响指定用户 |

---

### 13.3 `put_clear_device_token` 的完整执行链与后果

[accounts.rs#L1422-L1441](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1422-L1441)

```rust
async fn put_clear_device_token(device_id: DeviceId, conn: DbConn) -> EmptyResult {
    if !CONFIG.push_enabled() { return Ok(()); }

    if let Some(device) = Device::find_by_uuid(&device_id, &conn).await {  // ① 随机取一条
        Device::clear_push_token_by_uuid(&device_id, &conn).await?;        // ② 清空所有行
        unregister_push_device(device.push_uuid.as_ref()).await?;           // ③ 注销一个 push_uuid
    }

    Ok(())
}
```

#### 逐步分析

**步骤 ①**：`find_by_uuid(&device_id)` → 随机取一条设备记录

假设数据：
| uuid | user_uuid | push_uuid | push_token |
|------|-----------|-----------|------------|
| `abc123` | `user_A` | `push-A` | `FCM_A` |
| `abc123` | `user_B` | `push-B` | `FCM_B` |

返回结果不确定——可能是 `user_A` 的记录，也可能是 `user_B` 的记录。

**步骤 ②**：`clear_push_token_by_uuid(&device_id)` → 批量清空

```sql
UPDATE devices SET push_token = NULL WHERE uuid = 'abc123'
```

**两条记录的 push_token 都被清空**，但只有 `find_by_uuid` 返回的那条记录的 `push_uuid` 被保留在了步骤 ① 的变量中。

**步骤 ③**：`unregister_push_device(device.push_uuid.as_ref())` → 仅注销一个

[push.rs#L138-L158](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/push.rs#L138-L158)：

```rust
pub async fn unregister_push_device(push_id: Option<&PushId>) -> EmptyResult {
    if !CONFIG.push_enabled() || push_id.is_none() { return Ok(()); }
    let auth_api_token = get_auth_api_token().await?;
    let auth_header = format!("Bearer {auth_api_token}");

    match make_http_request(
        Method::POST,
        &format!("{}/push/delete/{}", CONFIG.push_relay_uri(), push_id.as_ref().unwrap()),
    )?
    .header(AUTHORIZATION, auth_header)
    .send()
    .await
    {
        Ok(r) => r,
        Err(e) => err!(format!("An error occurred during device unregistration: {e}")),
    };
    Ok(())
}
```

**只注销了一个 `push_uuid`**（步骤 ① 随机取到的那条），另一个 `push_uuid` 仍在 Relay 上注册。

---

### 13.4 多账号场景下的具体后果

#### 场景：手机上登录了两个账号，调用 clear-token

```
初始状态:
  Row 1: (abc123, user_A, push-A, FCM_A)  ← Relay 已注册 push-A
  Row 2: (abc123, user_B, push-B, FCM_B)  ← Relay 已注册 push-B

调用: PUT /devices/identifier/abc123/clear-token

步骤 ① find_by_uuid("abc123")
  → 返回 Row 1 (user_A 的记录，假设排序如此)

步骤 ② clear_push_token_by_uuid("abc123")
  → UPDATE devices SET push_token = NULL WHERE uuid = 'abc123'
  → Row 1: push_token = NULL  ← user_A 的 token 清空
  → Row 2: push_token = NULL  ← user_B 的 token 也被清空！

步骤 ③ unregister_push_device(push-A)
  → POST https://push.bitwarden.com/push/delete/push-A
  → push-A 从 Relay 注销 ✓
  → push-B 仍在 Relay 注册 ✗ ← 悬空！
```

#### 后果矩阵

| 用户 | push_token | push_uuid | Relay 状态 | 推送能否到达 |
|------|-----------|-----------|-----------|------------|
| user_A | **已清空** | push-A（已注销） | 已注销 | ✗ 无法推送 |
| user_B | **已清空** | push-B（未注销） | **悬空注册** | ✗ 无法推送（token 已清空，check 不通过） |

#### 悬空注册的具体危害

1. **Relay 侧残留**：`push-B` 仍在 Bitwarden Push Relay 上注册，但 Vaultwarden 侧已无对应 push_token
2. **无法接收推送**：因为 `check_user_has_push_device` 查询 `push_token IS NOT NULL`，token 清空后检查不通过，不会发送推送
3. **无法自动恢复**：user_B 需要重新登录并上报 token 才能恢复推送功能
4. **无主动清理**：Relay 侧的悬空注册不会过期，只能等用户重新登录时 `register_push_device` 覆盖

---

### 13.5 POST / PUT 双入口分析

[accounts.rs#L1390-L1393](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1390-L1393) 与 [accounts.rs#L1443-L1447](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/core/accounts.rs#L1443-L1447)

```rust
// Token 上报：POST 和 PUT 都指向同一实现
#[post("/devices/identifier/<device_id>/token", data = "<data>")]
async fn post_device_token(device_id: DeviceId, data: Json<PushToken>, headers: Headers, conn: DbConn) -> EmptyResult {
    put_device_token(device_id, data, headers, conn).await
}

#[put("/devices/identifier/<device_id>/token", data = "<data>")]
async fn put_device_token(device_id: DeviceId, data: Json<PushToken>, headers: Headers, conn: DbConn) -> EmptyResult {
    // ... 实际逻辑
}

// Token 清理：POST 和 PUT 也都指向同一实现
#[put("/devices/identifier/<device_id>/clear-token")]
async fn put_clear_device_token(device_id: DeviceId, conn: DbConn) -> EmptyResult {
    // ... 实际逻辑
}

#[post("/devices/identifier/<device_id>/clear-token")]
async fn post_clear_device_token(device_id: DeviceId, conn: DbConn) -> EmptyResult {
    put_clear_device_token(device_id, conn).await
}
```

#### 双入口的设计意图

代码注释说明：

```rust
// On upstream server, both PUT and POST are declared.
// Implementing the POST method in case it would be useful somewhere
```

- 上游 Bitwarden Server 同时声明了 PUT 和 POST
- Vaultwarden 为兼容性也提供了两种 HTTP 方法
- **两者行为完全一致**，POST 只是 PUT 的委托转发

#### 安全影响

两个入口共享同一实现，意味着：
- `POST /devices/identifier/<id>/clear-token` 同样无认证
- `PUT /devices/identifier/<id>/clear-token` 同样无认证
- 攻击面是双倍的，但实际风险不变（都走同一函数）

---

### 13.6 与 Admin deauth 的对比：正确的做法

[admin.rs#L463-L480](file:///d:/fz/0601/solo-dogfeeding/code/11-vaultwarden/src/api/admin.rs#L463-L480) 提供了一个**正确的多设备注销实现**：

```rust
async fn deauth_user(user_id: UserId, _token: AdminToken, conn: DbConn, nt: Notify<'_>) -> EmptyResult {
    let mut user = get_user_or_404(&user_id, &conn).await?;
    nt.send_logout(&user, None, &conn).await;

    if CONFIG.push_enabled() {
        for device in Device::find_push_devices_by_user(&user.uuid, &conn).await {
            match unregister_push_device(device.push_uuid.as_ref()).await {
                Ok(r) => r,
                Err(e) => error!("Unable to unregister devices from Bitwarden server: {e}"),
            }
        }
    }

    Device::delete_all_by_user(&user.uuid, &conn).await?;
    // ...
}
```

**关键差异**：

| 维度 | `put_clear_device_token` | `deauth_user`（Admin） |
|------|-------------------------|----------------------|
| 查询范围 | `find_by_uuid` → 跨用户 | `find_push_devices_by_user` → 按用户 |
| 遍历所有设备 | 否，只取一条 | 是，逐一处理 |
| Relay 注销 | 只注销一个 push_uuid | **遍历注销所有** push_uuid |
| 认证 | 无 | AdminToken |
| 影响范围 | 可能波及其他用户 | 仅影响目标用户 |

---

### 13.7 问题总结与修复方向

#### 三个叠加的问题

1. **`find_by_uuid` 跨用户取单条** → 随机取到某个用户的设备
2. **`clear_push_token_by_uuid` 跨用户批量清空** → 所有用户在该设备上的 token 都被清空
3. **只注销一个 push_uuid** → 其他用户的 push_uuid 在 Relay 上悬空

#### 修复方向

**方案 A：精确清理（推荐）**
- 改用 `find_by_uuid_and_user` 定位设备
- 改用带 `user_uuid` 条件的 `clear_push_token`
- 仅清理当前用户的 push_token 和 push_uuid

**方案 B：全局清理（当前行为的修正版）**
- `find_by_uuid` 改为查询所有匹配行（返回 `Vec<Self>`）
- 遍历所有匹配设备，逐一注销 push_uuid
- 确保所有 Relay 注册都被清理

**方案 C：简化方案**
- 补充 `Headers` 认证守卫
- 让清理 Token API 与设置 Token API 使用相同的定位逻辑
- 最小改动，最大一致性
