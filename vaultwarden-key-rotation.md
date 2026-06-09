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
- 通过 [update_send_from_data](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) 更新 send 的 `akey` 和加密数据

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
