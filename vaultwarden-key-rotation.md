# Vaultwarden 密钥轮换（Cipher/Master Key Rotation）深度分析

## 一、概述

密钥轮换是 Bitwarden/Vaultwarden 的核心安全机制，允许用户在不丢失数据的前提下更换主密码。整个轮换流程完全在客户端完成加密/解密操作，服务端仅存储和转发加密数据。

**入口端点**：`POST /accounts/key-management/rotate-user-account-keys`  
**处理函数**：[post_rotatekey](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L797-L917)

---

## 二、核心密钥体系

### 2.1 密钥层级结构

```
用户输入密码
    │
    ▼
Master Key (主密钥，客户端推导，不上传)
    │
    ├──► 用户认证哈希（master_key_authentication_hash）
    │       用于服务端密码验证
    │
    └──► 用户密钥（User Key / akey）
            │
            ├──► 账户私钥（Account Private Key）
            │       由 User Key 加密存储：user_key_encrypted_account_private_key
            │
            ├──► 各 Cipher 的对称密钥（Cipher Key，每个 cipher 独立）
            │       由 User Key 加密存储：cipher.key
            │           │
            │           └──► Cipher 实际数据（name/notes/fields/login/card/...）
            │                   由 Cipher Key 加密
            │
            ├──► Folder 名称（由 User Key 加密）
            ├──► Send 数据（由 Send Key 加密，Send Key 由 User Key 加密）
            ├──► Emergency Access 密钥（key_encrypted）
            ├──► Organization Reset Password Key
            └──► Attachment 密钥（akey）
```

### 2.2 服务端存储的密钥相关字段

| 模型 | 字段 | 说明 | 代码位置 |
|------|------|------|----------|
| User | `akey` | 由主密钥加密的 User Key | [user.rs:48](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs#L48-L48) |
| User | `private_key` | 由 User Key 加密的账户私钥 | [user.rs:49](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs#L49-L49) |
| User | `public_key` | 账户公钥（明文） | [user.rs:50](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs#L50-L50) |
| Cipher | `key` | 由 User Key 加密的 Cipher Key | [cipher.rs:43](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L43-L43) |
| Cipher | `name/notes/data/...` | 由 Cipher Key 加密的实际数据 | [cipher.rs:53-59](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L53-L59) |
| Send | `akey` | Send 的加密密钥 | [send.rs:34](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs#L34-L34) |
| EmergencyAccess | `key_encrypted` | 紧急访问加密密钥 | [emergency_access.rs:24](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs#L24-L24) |
| Membership | `reset_password_key` | 组织重置密码密钥 | [organization.rs:58](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs#L58-L58) |
| Membership | `akey` | 组织成员加密密钥 | [organization.rs:55](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs#L55-L55) |
| Attachment | `akey` | 附件加密密钥 | [attachment.rs:32](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/attachment.rs#L32-L32) |

---

## 三、密钥轮换的完整流程

### 3.1 请求数据结构

客户端发起轮换请求时提交的 [KeyData](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L677-L682)：

```rust
struct KeyData {
    account_unlock_data: RotateAccountUnlockData,  // 解锁数据
    account_keys: RotateAccountKeys,               // 新的账户密钥
    account_data: RotateAccountData,               // 重加密后的用户数据
    old_master_key_authentication_hash: String,    // 旧主密钥认证哈希（验证用户身份）
}
```

**子结构**：

- [RotateAccountUnlockData](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L686-L690)：包含紧急访问、主密码、组织恢复的解锁数据
- [RotateAccountKeys](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L706-L709)：新的 User Key 加密的私钥 + 公钥
- [RotateAccountData](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L713-L717)：重加密后的 ciphers、folders、sends

### 3.2 服务端执行步骤

**Step 1：身份验证** — [accounts.rs:802-804](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L802-L804)
```rust
if !headers.user.check_valid_password(&data.old_master_key_authentication_hash) {
    err!("Invalid password")
}
```

**Step 2：数据完整性验证** — [validate_keydata](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L719-L795)

服务器逐一检查：
- KDF 参数和邮箱**不允许**在轮换时变更
- 非对称密钥对（公钥）**不允许**变更
- **所有**个人 ciphers 必须包含在轮换请求中（通过 ID 超集校验）
- **所有** folders 必须包含在轮换请求中
- **所有** emergency access 密钥必须包含在轮换请求中
- **所有**已设置的组织 reset password keys 必须包含在轮换请求中
- **所有** sends 必须包含在轮换请求中

**Step 3：更新 Folders** — [accounts.rs:834-846](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L834-L846)
- 用新 User Key 重新加密的 folder name 覆盖旧值
- Folder 的 `id` 为 Option 类型，过滤 null（兼容 Bitwarden 客户端 bug [#8453](https://github.com/bitwarden/clients/issues/8453)）

**Step 4：更新 Emergency Access Keys** — [accounts.rs:848-858](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L848-L858)
- 更新 `key_encrypted` 字段

**Step 5：更新 Organization Reset Password Keys** — [accounts.rs:860-870](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L860-L870)
- 按 organization_id 匹配并更新 `reset_password_key`

**Step 6：更新 Sends** — [accounts.rs:872-879](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L872-L879)
- 通过 [update_send_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L609-L665) 更新 send 的 `akey` 和加密数据

#### 3.2.1 File Send vs Text Send 在密钥轮换中的不对称更新

Send 分为 Text Send（类型 0）和 File Send（类型 1）。两者都会在轮换时更新 `send.akey`、`name`、`notes`、删除时间等通用字段，但 `sends.data` 的更新路径不同。

**SendData 请求结构** — [sends.rs:72-91](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L72-L91)：
```rust
pub struct SendData {
    r#type: i32,            // 0 = Text, 1 = File
    key: String,            // 用新 User Key 重新加密后的 Send Key
    name: String,
    notes: Option<String>,
    text: Option<Value>,    // Text Send 专用
    file: Option<Value>,    // File Send 专用；更新时通常为 null
    pub id: Option<SendId>, // key rotation 用来定位已有 Send
}
```

**update_send_from_data 的关键分支** — [sends.rs:631-641](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L631-L641)：
```rust
// When updating a file Send, we receive nulls in the File field, as it's immutable,
// so we only need to update the data field in the Text case
if data.r#type == SendType::Text as i32 {
    let data_str = if let Some(mut d) = data.text {
        d.as_object_mut().and_then(|d| d.remove("response"));
        serde_json::to_string(&d)?
    } else {
        err!("Send data not provided");
    };
    send.data = data_str;
}

send.name = data.name;
send.akey = data.key;
send.notes = data.notes;
```

结论：

1. **Text Send 会重写 `sends.data`**。Text 类型的数据体来自请求里的 `text` 字段，服务端去掉 `response` 后重新序列化并覆盖 `send.data`。
2. **File Send 不重写 `sends.data`**。注释明确说明更新 File Send 时收到的 `file` 字段为 null，因为文件信息不可变，所以服务端只在 Text 分支更新 data。
3. **两种类型都会更新 `send.akey`**。轮换改变的是“用新 User Key 包裹 Send Key 的密文”，不是让服务端解密或重新加密文件内容。
4. **服务端不判断 file/text 内容是否加密**。`sends.data` 只是客户端提交的 JSON 字符串；对 File Send 来说，服务端保留原有 data，并继续通过 [Send::to_json](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs#L155-L156) 按 atype 输出到 `file` 字段。

**to_json 序列化时的对应关系**：
```rust
"text": if self.atype == SendType::Text as i32 { Some(&data) } else { None },
"file": if self.atype == SendType::File as i32 { Some(&data) } else { None },
```
同一个 `sends.data` 字段会根据 atype 被映射成响应里的 `text` 或 `file`，但轮换更新阶段只有 Text Send 会把请求体重新写入 `sends.data`。

**Step 7：更新 Ciphers（核心步骤）** — [accounts.rs:881-894](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L881-L894)
- 仅处理个人 ciphers（`organization_id.is_none()`）
- 调用 [update_cipher_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L395-L576)，传入 `UpdateType::None`
- **关键**：每个 cipher 的 `key` 字段（Cipher Key）和所有加密数据字段都被新 User Key 重新加密后的值覆盖

Cipher 更新的核心代码 — [ciphers.rs:526-531](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L526-L531)：
```rust
cipher.key = data.key;              // 新 User Key 加密的 Cipher Key
cipher.name = data.name;            // 新 Cipher Key 加密的名称
cipher.notes = data.notes;          // 新 Cipher Key 加密的备注
cipher.fields = ...;                // 新 Cipher Key 加密的字段
cipher.data = type_data.to_string(); // 新 Cipher Key 加密的类型数据（login/card/...）
cipher.password_history = ...;      // 新 Cipher Key 加密的密码历史
```

**Step 8：更新用户主密钥和私钥** — [accounts.rs:896-907](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L896-L907)
```rust
user.private_key = Some(data.account_keys.user_key_encrypted_account_private_key);
user.set_password(
    &data.account_unlock_data.master_password_unlock_data.master_key_authentication_hash,
    Some(data.account_unlock_data.master_password_unlock_data.master_key_encrypted_user_key),
    true,  // reset_security_stamp = true
    None,
    &conn,
).await?;
```

[set_password](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs#L192-L215) 内部完成：
1. 更新 `password_hash`（新主密钥认证哈希）
2. 更新 `akey`（新主密钥加密的 User Key）
3. **重置 `security_stamp`** — 这会触发所有其他设备的 refresh token 失效

**Step 9：保存用户并登出其他设备** — [accounts.rs:909-916](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L909-L916)
```rust
let save_result = user.save(&conn).await;
nt.send_logout(&user, Some(&headers.device), &conn).await; // 排除当前设备
```

---

## 四、对加密数据的影响

### 4.1 被重新加密的所有数据

| 数据类型 | 加密密钥变化 | 服务端存储字段 |
|----------|-------------|---------------|
| **User Key** | Master Key（旧）→ Master Key（新） | `users.akey` |
| **Account Private Key** | User Key（不变），但因 User Key 本身被重新加密而间接变化 | `users.private_key` |
| **每个 Cipher 的 Cipher Key** | User Key（旧）→ User Key（新） | `ciphers.key` |
| **Cipher 数据** | Cipher Key（不变），客户端解后用同一 Cipher Key 重加密 | `ciphers.name`, `notes`, `fields`, `data`, `password_history` |
| **Folder 名称** | User Key（旧）→ User Key（新） | `folders.name` |
| **Send Key** | User Key（旧）→ User Key（新） | `sends.akey` |
| **Send 数据** | Send Key（不变） | `sends.name`, `notes`, `text`, `file` |
| **Emergency Access Key** | User Key（旧）→ User Key（新） | `emergency_access.key_encrypted` |
| **Org Reset Password Key** | User Key（旧）→ User Key（新） | `users_organizations.reset_password_key` |
| **Attachment Key** | Cipher Key（不变），因 Cipher Key 不变间接不变 | `attachments.akey` |

### 4.2 关键洞察

1. **两层加密架构是轮换的基础**：
   - Master Key → User Key → Cipher Key → Cipher Data
   - 轮换时只需重新加密"密钥的密钥"（User Key 层），最底层的数据加密密钥（Cipher Key）本身不变
   - 但 Cipher Data 仍需客户端解密后用同一 Cipher Key 重新加密（实际上密钥没变，只是客户端重新打包）

2. **Cipher Key 字段是 2023.10 新增的**：
   - 数据库迁移：[2023-10-21-221242_add_cipher_key](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/migrations/sqlite/2023-10-21-221242_add_cipher_key/up.sql)
   - 迁移内容：`ALTER TABLE ciphers ADD COLUMN "key" TEXT;`
   - 旧版本 cipher 无此字段（值为 NULL），新版客户端使用每个 cipher 独立的密钥

3. **非对称密钥对（公钥/私钥）在轮换中保持不变**：
   - 公钥直接校验必须与现有值一致：[accounts.rs:736-738](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L736-L738)
   - 私钥只是用新 User Key 重新加密，密钥本身没变
   - 这保证了组织共享、紧急访问等依赖非对称加密的功能不受影响

---

## 五、对同步状态的影响

### 5.1 Revision Date 机制

Vaultwarden 使用 `updated_at` / `revision_date` 时间戳作为客户端增量同步的依据：

- **User.updated_at**：用户维度的全局修订版本
- **Cipher.updated_at**：单个密码条目修订版本
- **Folder.updated_at**：单个文件夹修订版本
- **Send.revision_date**：单个 Send 修订版本

每次保存都会自动更新修订时间：
- Cipher 保存时更新：[cipher.rs:442-444](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L442-L444)
- Folder 保存时更新：[folder.rs:75-77](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/folder.rs#L75-L77)
- 同时级联更新 User 的修订版本：通过 [User::update_uuid_revision](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs#L351-L355)

### 5.2 轮换期间的同步策略

**核心设计决策：使用 UpdateType::None 抑制实时推送通知**

在密钥轮换的 cipher 更新循环中：[accounts.rs:889-892](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L889-L892)
```rust
// Prevent triggering cipher updates via WebSockets by settings UpdateType::None
// The user sessions are invalidated because all the ciphers were re-encrypted and thus triggering an update could cause issues.
// We force the users to logout after the user has been saved to try and prevent these issues.
update_cipher_from_data(saved_cipher, cipher_data, &headers, None, &conn, &nt, UpdateType::None).await?;
```

**原因**：
1. 如果每个 cipher 更新都发送 WebSocket 推送，其他设备会收到大量零散更新
2. 但此时其他设备还持有旧的 Master Key，无法解密新数据，可能导致客户端状态混乱
3. 更好的策略是：完成所有更新后，强制其他设备重新登录 → 全量同步


### 5.2.1 Send 密钥轮换的同步机制深度解析

Send 的更新与 Cipher 采用完全相同的同步抑制策略，但具体实现位于 [update_send_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L609-L665)。

**调用链对比**：

```
密钥轮换入口 [accounts.rs:878]
    │
    └── update_send_from_data(send, send_data, &headers, &conn, &nt, UpdateType::None)
            │
            ├── 1. 更新 send.akey = data.key         // Send Key 用新 User Key 重新加密
            ├── 2. 更新 send.name/notes/data...      // 重新加密的数据
            │
            ├── 3. send.save(conn)                   // [send.rs:197-L229]
            │       │
            │       ├── send.update_users_revision(conn)  // [send.rs:252-L261]
            │       │     └── User::update_uuid_revision(user_uuid, conn)
            │       │           └── UPDATE users SET updated_at = NOW()
            │       │
            │       └── send.revision_date = NOW()
            │
            └── 4. if ut != UpdateType::None          // [sends.rs:661-L663]
                     └── 条件为 false，跳过 nt.send_send_update(...)
```

**关键洞察：两个层次的同步静默**

1. **实时推送层被抑制**：
   - `UpdateType::None` 使 [sends.rs:661](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L661-L663) 的条件判断直接跳过 `nt.send_send_update()`。
   - WebSocket 不会向其他设备推送 SyncSendUpdate 消息，Push 通知也不会发送到移动设备。
   - 这与 Cipher 更新时的处理完全对称，Cipher 也在相同的 `if ut != UpdateType::None` 分支后才推送。

2. **修订时间戳仍然更新**：
   - `send.save()` 内部会无条件执行 `self.update_users_revision(conn)`，更新 `User.updated_at`。
   - 同一个保存流程还会把 `self.revision_date` 更新为当前时间，保留 Send 自身的修订版本。
   - 其他设备重新登录后做增量同步时，可以通过用户级和 Send 级时间戳发现 Send 已经变化，再拉取重新加密后的数据。

如果连时间戳也不更新，其他设备重新登录后可能基于上次同步时间做增量拉取，从而漏掉 Send 的变更，最终拿不到重新加密后的 Send 数据。因此轮换流程只抑制实时推送，不抑制修订时间戳。

| 同步层面 | Cipher [update_cipher_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L545-L574) | Send [update_send_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L660-L663) |
|---------|------|------|
| 实时推送抑制 | `if ut != UpdateType::None` | `if ut != UpdateType::None` |
| 修订时间戳更新 | `cipher.save()` 内部更新用户修订时间和 cipher 自身时间 | `send.save()` 内部更新用户修订时间和 Send 自身 `revision_date` |
| 增量同步可见性 | 重新登录后可通过修订时间发现 | 重新登录后可通过修订时间发现 |

因此“Send 更新没有触发同步”只在实时推送层成立；增量同步依赖的时间戳仍然被正确更新。

### 5.3 完整的同步失效链

```
密钥轮换开始
    │
    ├──► 逐个更新 Cipher（UpdateType::None，不推送）
    ├──► 更新 Folder（触发 User.updated_at 更新）
    ├──► 更新 Send
    ├──► 更新 Emergency Access / Reset Password Keys
    │
    ├──► User.set_password()
    │       │
    │       ├──► 更新 password_hash
    │       ├──► 更新 akey（User Key）
    │       └──► 重置 security_stamp
    │               │
    │               └──► Device::rotate_refresh_tokens_by_user()
    │                       使所有 refresh token 失效
    │
    └──► nt.send_logout(user, Some(current_device))
            │
            ├──► WebSocket: 向用户的所有设备（除当前设备）推送 LogOut 消息
            └──► Push: 向移动设备推送登出通知
```

### 5.4 UpdateType 枚举

[notifications.rs:622-654](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/notifications.rs#L622-L654) 定义了所有同步事件类型：

| 值 | 类型 | 说明 |
|----|------|------|
| 0 | SyncCipherUpdate | 单个密码更新 |
| 1 | SyncCipherCreate | 新建密码 |
| 5 | SyncVault | 整个 vault 同步（如导入完成） |
| 6 | SyncOrgKeys | 组织密钥同步 |
| 11 | LogOut | 强制登出 |
| 100 | None | 不触发任何推送（密钥轮换时使用） |

---

## 六、兼容边界与向后兼容策略

### 6.1 Cipher Key 字段的兼容性

**问题**：`ciphers.key` 字段是 2023.10 新增的，历史数据可能为 NULL。

**处理方式**：
- 模型定义为 `Option<String>`：[cipher.rs:43](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L43-L43)
- CipherData 请求体中也是 `Option<String>`：[ciphers.rs:261](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L261-L261)
- 序列化到 JSON 时直接输出，为 NULL 则表示该 cipher 使用旧的"全局 User Key 直接加密"模式：[cipher.rs:345](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L345-L345)


### 6.1.1 旧 cipher.key (NULL) 在密钥轮换中的完整兼容路径

这是密钥轮换兼容性的核心：2023.10 之前创建、没有独立 Cipher Key 的旧密码条目，在轮换时仍要保持可解密。

#### 两种加密模式的本质区别

```
模式 A（旧模式，cipher.key = NULL）
  User Key → 直接加密 Cipher Data（name/notes/data/...）
  没有中间层，每个 cipher 没有独立密钥

模式 B（新模式，cipher.key = Some("..."))
  User Key → 加密 Cipher Key（存储在 cipher.key 字段）
  Cipher Key → 加密 Cipher Data（name/notes/data/...）
  每个 cipher 有独立的对称密钥
```

#### 兼容的五层防线

**第一层：数据模型层使用 Option<String>**

服务端三个关键位置都允许 NULL 合法存在：

| 位置 | 定义 | 代码位置 |
|------|------|----------|
| 数据库模型 | `pub key: Option<String>` | [cipher.rs:43](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L43-L43) |
| 请求体反序列化 | `key: Option<String>` | [ciphers.rs:261](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L261-L261) |
| JSON 序列化输出 | `json!({"key": self.key})` 可输出 null | [cipher.rs:345](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs#L345-L345) |

这意味着旧 cipher 从数据库读出时是 `None`，返回给客户端时是 `"key": null`；客户端轮换请求不传 key 或传 null，服务端反序列化后仍是 `None`。

**第二层：校验层不检查 key 字段值**

[validate_keydata](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L740-L751) 只做 cipher ID 的超集校验：

```rust
let existing_cipher_ids = existing_ciphers.iter().map(|c| &c.uuid).collect();
let provided_cipher_ids = data.account_data.ciphers.iter()
    .filter(|c| c.organization_id.is_none())
    .filter_map(|c| c.id.as_ref())
    .collect();
if !provided_cipher_ids.is_superset(&existing_cipher_ids) {
    err!("All existing ciphers must be included in the rotation")
}
```

这段逻辑只要求所有个人 cipher 都出现在请求里，不校验请求是否带 key、key 是否与数据库原值匹配，也不检查 key 的加密格式。`Cipher::validate_cipher_data` 也只校验 notes 大小和 password_history 的 null 值，不检查 `cipher.key`。

**第三层：更新层是对称赋值**

[update_cipher_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L526-L531) 直接把请求值写回：

```rust
cipher.key = data.key;              // 旧模式下 data.key = None → cipher.key 保持 None
cipher.name = data.name;            // 重新加密后的数据
cipher.notes = data.notes;
cipher.fields = ...;
cipher.data = type_data.to_string();
cipher.password_history = ...;
```

原来 `cipher.key` 是 None 时，只要客户端按旧模式提交 None，服务端就继续保存 NULL；如果客户端在轮换时提交新的加密 Cipher Key，服务端也会无感地把条目升级到新模式。

**第四层：不同客户端版本都能落到同一套 Option 语义**

| 客户端行为 | 服务端接收结果 | 保存结果 |
|-----------|---------------|----------|
| 不发送 key 字段 | `data.key = None` | `cipher.key = NULL` |
| 发送 `"key": null` | `data.key = None` | `cipher.key = NULL` |
| 发送 `"key": "enc(...)"` | `data.key = Some(...)` | `cipher.key = Some(...)` |

**第五层：加密逻辑由客户端负责**

服务端不参与加密或解密，也不校验加密数据能否被旧模式或新模式解开。只要客户端能正确完成“解密旧值 → 按目标模式重新加密 → 提交给服务端”，服务端就按请求体原样存储。这个零信任边界让旧条目可以继续保持 NULL，也可以由新客户端在轮换时静默升级为独立 Cipher Key 模式。

```
DB: cipher.key = NULL
  ├── 客户端未发送 key      → data.key = None      → 继续保存 NULL
  ├── 客户端发送 key = null → data.key = None      → 继续保存 NULL
  └── 客户端发送 enc(ck)    → data.key = Some(...) → 升级为新模式
```

### 6.2 旧客户端与新服务端的兼容

**无 cipher.key 的旧数据在新服务端上**：
- 服务端透明返回 NULL 值
- 客户端根据是否存在 key 字段决定使用何种解密路径

### 6.3 轮换时的边界校验

[validate_keydata](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L719-L795) 强制了以下边界：

1. **禁止同时变更 KDF 参数**：
   ```rust
   if user.client_kdf_type != ... || user.client_kdf_iter != ... || user.email != ... {
       err!("Changing the kdf variant or email is not supported during key rotation");
   }
   ```
   理由：KDF 变更会改变 Master Key 的推导方式，与轮换逻辑耦合度太高，拆分为独立操作。

2. **禁止变更非对称密钥对**：
   ```rust
   if user.public_key.as_ref() != Some(&data.account_keys.account_public_key) {
       err!("Changing the asymmetric keypair is not possible during key rotation")
   }
   ```
   理由：组织共享、紧急访问、Sends 等均依赖公私钥体系，更换会导致大范围数据不可解密。

3. **全量数据覆盖**：必须包含所有现有的 ciphers、folders、sends、emergency access、reset password keys，确保轮换后没有遗留的用旧密钥加密的数据。

4. **仅处理个人密码**：组织密码（`organization_id.is_some()`）在轮换流程中被跳过，由组织密钥轮换机制处理。

#### 6.3.1 id 缺失与额外条目的边界条件

轮换接口对全量数据覆盖的检查分成“校验阶段”和“更新阶段”。这里的细节是：Ciphers、Sends、Folders 的 id 都是 `Option`，但后续处理方式不一样。

**id 字段定义**：

| 数据类型 | id 字段 | 代码位置 |
|----------|--------|----------|
| Cipher | `id: Option<CipherId>` | [ciphers.rs:256](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs#L256) |
| Send | `pub id: Option<SendId>` | [sends.rs:90](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L90) |
| Folder | `id: Option<FolderId>` | [accounts.rs:657](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L657) |
| Emergency Access | `id: EmergencyAccessId`，不可缺失 | [accounts.rs:664](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L664) |
| Reset Password Key | `organization_id: OrganizationId`，不可缺失 | [accounts.rs:671](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L671) |

**校验阶段：只检查已有 id 是否被覆盖**

以 Sends 为例，[validate_keydata](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L787-L792) 使用 `filter_map` 收集请求里的 id：
```rust
let existing_send_ids = existing_sends.iter().map(|s| &s.uuid).collect::<HashSet<&SendId>>();
let provided_send_ids = data.account_data.sends.iter().filter_map(|s| s.id.as_ref()).collect::<HashSet<&SendId>>();
if !provided_send_ids.is_superset(&existing_send_ids) {
    err!("All existing sends must be included in the rotation")
}
```

这个阶段有两个后果：

- `id = None` 的条目会被过滤掉；只要其他条目已经覆盖数据库里的所有既有 id，校验仍可通过。
- 额外 id 不会被挡住；`is_superset` 只要求“包含全部已有 id”，不要求请求集合与数据库集合完全相等。

Ciphers 的校验逻辑相同，也使用请求 id 集合对既有 id 集合做超集检查。

**更新阶段：Ciphers/Sends 使用 unwrap，Folders 用 if let**

Sends 更新循环 [accounts.rs:873-L879](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L873-L879)：
```rust
for send_data in data.account_data.sends {
    let Some(send) = existing_sends.iter_mut().find(|s| &s.uuid == send_data.id.as_ref().unwrap()) else {
        err!("Send doesn't exist")
    };
    update_send_from_data(send, send_data, ...)
}
```

Ciphers 更新循环也直接对 `cipher_data.id.as_ref().unwrap()` 做匹配。相比之下，Folders 先用 `if let Some(folder_id) = folder_data.id` 判断，缺 id 的条目会被安全跳过。

| 数据类型 | 缺失 id（校验阶段） | 缺失 id（更新阶段） | 额外不存在 id（更新阶段） |
|----------|-------------------|-------------------|----------------------|
| Ciphers | 可能通过：`filter_map` 会过滤 None | `unwrap()` 触发 panic | 返回 `Cipher doesn't exist` |
| Sends | 可能通过：`filter_map` 会过滤 None | `unwrap()` 触发 panic | 返回 `Send doesn't exist` |
| Folders | 可能通过：`filter_map` 会过滤 None | `if let` 安全跳过 | 返回 `Folder doesn't exist` |
| Emergency Access | id 非 Option，反序列化阶段就必须存在 | 不适用 | 返回业务错误 |
| Reset Password Key | organization_id 非 Option，反序列化阶段就必须存在 | 不适用 | 返回业务错误 |

因此，`id = None` 的 Cipher/Send 条目是一个校验-更新不一致的边界：校验阶段可能因为 `filter_map` 被忽略，更新阶段却会在 `unwrap()` 处触发 panic。额外但不存在的 id 则不会 panic，会走到明确的业务错误。

### 6.4 事务与部分失败问题

代码中有明确的 TODO 注释：[accounts.rs:799](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L799-L799) 和 [accounts.rs:814](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L814-L814)
```rust
// TODO: See if we can wrap everything within a SQL Transaction. If something fails it should revert everything.
// TODO: Ideally we'd do everything after this point in a single transaction.
```

**当前风险**：
- Folders 更新成功但后续 Ciphers 更新失败 → 部分数据用新密钥加密，部分用旧密钥
- 最终用户保存成功 → security_stamp 重置 → 所有设备登出 → 但数据不完整
- 客户端必须重新提交完整的轮换请求，已更新过的条目会被再次覆盖（幂等操作，风险可控）

### 6.5 Folder ID 的 null 兼容

[accounts.rs:653-655](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L653-L655) 中的特殊处理：
```rust
// There is a bug in 2024.3.x which adds a `null` item.
// To bypass this we allow a Option here, but skip it during the updates
// See: https://github.com/bitwarden/clients/issues/8453
```
服务端使用 `Option<FolderId>` 并在更新时过滤 `None` 值，兼容有 bug 的客户端版本。

---

## 七、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 轮换入口函数 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L797-L917 |
| 请求数据校验 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L719-L795 |
| Cipher 数据更新 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs) | L395-L576 |
| User.set_password | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs) | L192-L215 |
| User 重置安全戳 | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs) | L217-L221 |
| Cipher JSON 序列化（含 key 字段） | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs) | L145-L412 |
| 通知登出 | [notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/notifications.rs) | L362-L381 |
| UpdateType 枚举 | [notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/notifications.rs) | L622-L654 |
| Cipher 模型（含 key 字段） | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs) | L35-L62 |
| CipherData 请求结构 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs) | L251-L301 |
| 数据库迁移：添加 cipher key | [2023-10-21-221242_add_cipher_key/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/migrations/sqlite/2023-10-21-221242_add_cipher_key/up.sql) | L1-L2 |
| 附件密钥模型 | [attachment.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/attachment.rs) | L27-L33 |
| 紧急访问密钥模型 | [emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs) | L23-L25 |
| 组织成员密钥模型 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs) | L49-L60 |
| Send 密钥模型 | [send.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs) | L30-L35 |
| Send 数据更新函数 | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L609-L665 |
| Send 推送抑制判断 | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L660-L663 |
| Send.save() 更新修订时间戳 | [send.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs) | L197-L199 |
| Send.update_users_revision() | [send.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs) | L252-L261 |
| Cipher 推送抑制判断 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/ciphers.rs) | L545-L574 |
| Cipher.validate_cipher_data（不校验 key） | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/cipher.rs) | L97-L140 |
| validate_keydata 的 cipher ID 超集校验 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L740-L751 |
| 轮换中 Send UpdateType::None 调用 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L872-L879 |
| 轮换中 Cipher UpdateType::None 调用 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L881-L894 |
| SendData 请求结构（含 text/file/id 字段） | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L70-L91 |
| update_send_from_data 中 Text/File 分支 | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L631-L641 |
| Send.to_json 中 text/file 序列化分支 | [send.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs) | L155-L156 |
| validate_keydata 的 send ID 校验（含 filter_map） | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L787-L792 |
| Sends 更新循环（含 unwrap 风险） | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L873-L879 |
| Ciphers 更新循环（含 unwrap 风险） | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L883-L892 |
| Folders 更新循环（if let 安全模式） | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L835-L845 |
| UpdateFolderData 结构（含 Option<FolderId>） | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L650-L659 |
| UpdateEmergencyAccessData 结构 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L661-L666 |
