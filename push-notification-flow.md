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
| 设备离线 | ✗ 连接断开，消息丢失 | ✓ Push Relay 负责离线缓存 |
| 应用后台 | ✗ WebSocket 可能被杀死 | ✓ 系统级推送拉起应用 |

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
