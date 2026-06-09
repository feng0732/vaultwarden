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

#### 3.2.3 紧急访问密钥（Emergency Access）的轮换范围与同步界限

**紧急访问的角色模型**

紧急访问涉及两个角色：
- **Grantor（授权人）**：发起密钥轮换的当前用户，将自己的 User Key 加密后授予他人紧急访问
- **Grantee（受让人）**：被授权可以紧急访问 grantor 账户的联系人

`key_encrypted` 字段存储的是 **Grantor 的 User Key 用 Grantee 的公钥加密后的密文** — [emergency_access.rs:24](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs#L24)。密钥轮换时，Grantor 的 User Key 发生变化，因此所有相关的 `key_encrypted` 都必须用新 User Key 重新加密。

**参与轮换的状态过滤（哪些条目参与轮换）**

加载现有紧急访问记录时使用了严格的状态过滤 — [accounts.rs:818](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L818)：
```rust
let mut existing_emergency_access =
    EmergencyAccess::find_all_confirmed_by_grantor_uuid(user_id, &conn).await;
```

[find_all_confirmed_by_grantor_uuid](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs#L363-L372) 的过滤条件：
```rust
.filter(emergency_access::status.ge(EmergencyAccessStatus::Confirmed as i32))
// status >= 2
```

**EmergencyAccessStatus 枚举** — [emergency_access.rs:133-139](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs#L133-L139)：

| 状态 | 值 | 是否参与轮换 | 原因 |
|------|---|------------|------|
| Invited | 0 | ❌ 不参与 | 仅发送了邀请，尚未被 grantee 接受，没有有效的 `key_encrypted` |
| Accepted | 1 | ❌ 不参与 | Grantee 已接受但 grantor 尚未确认，没有设置 `key_encrypted` |
| Confirmed | 2 | ✅ 参与 | 完整建立，`key_encrypted` 已设置 |
| RecoveryInitiated | 3 | ✅ 参与 | Grantee 发起了紧急恢复请求，`key_encrypted` 仍然有效 |
| RecoveryApproved | 4 | ✅ 参与 | 紧急恢复已批准，`key_encrypted` 仍然有效 |

**轮换时的字段更新范围**

[accounts.rs:856-857](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L856-L857)：
```rust
saved_emergency_access.key_encrypted = Some(emergency_access_data.key_encrypted);
saved_emergency_access.save(&conn).await?;
```

轮换时只更新一个字段：
- ✅ `key_encrypted`：用新 User Key 重新加密的密文

**明确不更新的字段**（保持原值）：
- ❌ `status`：轮换不会改变紧急访问的状态
- ❌ `wait_time_days`：等待时间不变
- ❌ `atype`：类型（View/Takeover）不变
- ❌ `grantee_uuid` / `email`：受让人不变
- ❌ `recovery_initiated_at` / `last_notification_at`：恢复相关时间戳不变
- ❌ `created_at`：创建时间不变

**同步界限：哪些用户的修订版本会被更新**

[EmergencyAccess::save](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs#L144-L146)：
```rust
pub async fn save(&mut self, conn: &DbConn) -> EmptyResult {
    User::update_uuid_revision(&self.grantor_uuid, conn).await;  // 只更新 grantor
    self.updated_at = Utc::now().naive_utc();
    ...
}
```

关键发现：**只更新 Grantor（授权人）的 `User.updated_at`，完全不更新 Grantee（受让人）的修订版本。**

```
Grantor User.updated_at  → ✅ 被更新（触发 grantor 自己其他设备的增量同步感知）
Grantee User.updated_at  → ❌ 不被更新
```

**关于 Grantee 端的同步缺失**

紧急访问密钥轮换时，Grantee 端不会收到任何通知：
- 没有 WebSocket 推送（`nt` 参数未传入 save，且 save 内不触发推送）
- Grantee 的 `User.updated_at` 不更新，增量同步也感知不到
- 没有 Push 通知

**Grantee 如何感知到变更？**

实际生效时机是在 Grantee 端发起紧急访问恢复时：
1. Grantee 从服务端拉取紧急访问记录
2. 使用自己的私钥解密 `key_encrypted` 获取 Grantor 的新 User Key
3. 用新 User Key 解密 Grantor 的密码条目

由于 `EmergencyAccess.updated_at` 自己在 save 时会被更新，Grantee 下次进入紧急访问页面时会拉到最新的 `key_encrypted`。但在 Grantee 的个人账户同步层面，这个变更不可见。

**Step 5：更新 Organization Reset Password Keys** — [accounts.rs:860-870](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L860-L870)
- 按 organization_id 匹配并更新 `reset_password_key`

#### 3.2.4 组织恢复密钥（Organization Reset Password）的轮换范围与同步界限

**组织恢复密钥的作用**

组织管理员（Admin/Owner）可以启用"账户恢复"功能。启用后，用户的 User Key 会用组织的公钥加密，存储在 `users_organizations.reset_password_key` 字段中 — [organization.rs:48](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs#L48)。当用户遗忘主密码时，组织管理员可以协助恢复。

密钥轮换时，User Key 发生变化，因此每个启用了恢复功能的组织成员关系都必须重新加密 `reset_password_key`。

**参与轮换的过滤条件（哪些成员关系参与轮换）**

两步过滤 — [accounts.rs:819-821](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L819-L821)：
```rust
let mut existing_memberships = Membership::find_by_user(user_id, &conn).await;
// We only rotate the reset password key if it is set.
existing_memberships.retain(|m| m.reset_password_key.is_some());
```

**第一步：find_by_user 的隐式过滤** — [organization.rs:1014-1022](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs#L1014-L1022)：
```rust
pub async fn find_by_user(user_uuid: &UserId, conn: &DbConn) -> Vec<Self> {
    conn.run(move |conn| {
        users_organizations::table
            .filter(users_organizations::user_uuid.eq(user_uuid))
            .load::<Self>(conn)  // 注意：没有按 status 过滤！
    }).await
}
```

`find_by_user` **不过滤状态**，意味着 Revoked(-1) 的成员关系也会被加载。

**第二步：retain 显式过滤**：
```rust
existing_memberships.retain(|m| m.reset_password_key.is_some());
```

只保留 `reset_password_key` 非 None 的成员关系。

**最终参与轮换的组合矩阵**：

| Membership 状态 | reset_password_key 存在 | 是否参与轮换 |
|----------------|----------------------|------------|
| Revoked (-1) | Yes | ✅ 参与 |
| Revoked (-1) | No | ❌ 不参与 |
| Invited (0) | Yes | ✅ 参与 |
| Invited (0) | No | ❌ 不参与 |
| Accepted (1) | Yes | ✅ 参与 |
| Accepted (1) | No | ❌ 不参与 |
| Confirmed (2) | Yes | ✅ 参与 |
| Confirmed (2) | No | ❌ 不参与 |

**反直觉之处**：已被组织 Revoked（撤销）的用户，如果 `reset_password_key` 仍然存在于数据库中，该字段仍然会参与密钥轮换并被重新加密。理论上被撤销的用户不应继续持有组织的恢复能力，但数据库层面没有在 revoke 时清理该字段。

**轮换时的字段更新范围**

[accounts.rs:868-869](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs#L868-L869)：
```rust
membership.reset_password_key = Some(reset_password_data.reset_password_key);
membership.save(&conn).await?;
```

轮换时只更新一个字段：
- ✅ `reset_password_key`：用新 User Key 重新加密的密文

**明确不更新的字段**（保持原值）：
- ❌ `status`：成员状态不变
- ❌ `atype`：成员类型（Owner/Admin/User/Manager）不变
- ❌ `access_all`：访问权限不变
- ❌ `akey`：组织密钥加密的成员密钥不变（组织密钥没有变化）
- ❌ `external_id`：外部 ID 不变
- ❌ `groups` / `collections`：关联关系不变

**同步界限：哪些用户的修订版本会被更新**

[Membership::save](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs#L739-L741)：
```rust
pub async fn save(&self, conn: &DbConn) -> EmptyResult {
    User::update_uuid_revision(&self.user_uuid, conn).await;  // 只更新当前用户自己
    ...
}
```

只更新 `self.user_uuid` — 也就是**发起密钥轮换的当前用户**的修订版本。

**不被更新的用户/实体**：
- ❌ 组织所有者（Owner）的修订版本
- ❌ 组织管理员（Admin）的修订版本
- ❌ 组织的 `Organization.revision_date`
- ❌ 其他组织成员的修订版本

```
Organization.revision_date       → ❌ 不更新
Organization Owner User.updated_at → ❌ 不更新
Organization Admin User.updated_at → ❌ 不更新
当前用户（轮换发起者）User.updated_at → ✅ 更新
```

**组织管理员如何感知到恢复密钥变更**：

组织管理员端不会自动感知到某个用户的恢复密钥发生了变化。当管理员真正发起账户恢复操作时：
1. 从服务端拉取该用户的 Membership
2. 使用组织私钥解密 `reset_password_key` 获取用户的新 User Key
3. 为用户重置主密码

由于 `membership.save()` 会更新用户自己的 `User.updated_at`，但不会更新组织或组织管理员的修订版本，因此组织侧的同步不会被触发。

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

#### 3.2.2 Send 的访问密码与限制字段在轮换中的处理

Send 除了加密数据外，还有一组与加密无关的"访问控制字段"。这些字段在密钥轮换中的处理逻辑需要单独理解。

**Send 的访问控制字段分类**：

| 字段 | 请求结构定义 | 数据库字段 | 是否与加密相关 | 处理模式 |
|------|-------------|-----------|---------------|---------|
| **访问密码** | `password: Option<String>` | `password_hash`, `password_salt`, `password_iter` | ❌ 无关（仅身份验证） | **条件覆盖** |
| **最大访问次数** | `max_access_count: Option<NumberOrString>` | `max_access_count` | ❌ 无关 | **空则清除，否则赋值** |
| **过期时间** | `expiration_date: Option<DateTime<Utc>>` | `expiration_date` | ❌ 无关 | **空则清除，否则赋值** |
| **删除时间** | `deletion_date: DateTime<Utc>` | `deletion_date` | ❌ 无关 | **必须提供，强制赋值** |
| **隐藏邮箱** | `hide_email: Option<bool>` | `hide_email` | ❌ 无关 | **直接赋值** |
| **是否禁用** | `disabled: bool` | `disabled` | ❌ 无关 | **直接赋值** |
| **已访问次数** | （请求中不存在） | `access_count` | ❌ 无关 | **保持不变** |

**update_send_from_data 后半段的完整逻辑** — [sends.rs:643-L658](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L643-L658)：
```rust
send.name = data.name;
send.akey = data.key;
send.deletion_date = data.deletion_date.naive_utc();   // 必须字段，强制覆盖
send.notes = data.notes;
send.max_access_count = match data.max_access_count {  // Option：None 则清除
    Some(m) => Some(m.into_i32()?),
    _ => None,
};
send.expiration_date = data.expiration_date.map(|d| d.naive_utc()); // Option：None 则清除
send.hide_email = data.hide_email;    // Option<bool>：None 也会被写回（清除隐藏邮箱）
send.disabled = data.disabled;        // 非 Option bool：必须显式 true/false

// 访问密码：只有显式传入才变更
if let Some(password) = data.password {
    send.set_password(Some(&password));
}
// ⚠️ data.password = None 时：保持旧密码不变（不会被清除）
```

##### 访问密码的"条件保留"机制

这是最容易误解的点：`SendData.password` 是 `Option<String>`，但语义不是"None 表示清除密码"，而是"None 表示**保持原有密码不变**"。

[Send::set_password](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs#L99-L113) 的实现：
```rust
pub fn set_password(&mut self, password: Option<&str>) {
    const PASSWORD_ITER: i32 = 100_000;
    if let Some(password) = password {
        // 设置密码：生成新的随机 salt + PBKDF2 哈希
        self.password_iter = Some(PASSWORD_ITER);
        let salt = crate::crypto::get_random_bytes::<64>().to_vec();
        let hash = crate::crypto::hash_password(password.as_bytes(), &salt, PASSWORD_ITER as u32);
        self.password_salt = Some(salt);
        self.password_hash = Some(hash);
    } else {
        // 清除密码：三个字段全部置 None
        self.password_iter = None;
        self.password_salt = None;
        self.password_hash = None;
    }
}
```

**三种场景对比**：

| 客户端发送 | 服务端行为 | 结果 |
|-----------|-----------|------|
| `"password": "newpass123"` | 调用 `set_password(Some("newpass123"))` | ✅ 密码被重设（生成新 salt + 新哈希） |
| `"password": null`（或不发送字段） | 不调用 `set_password`（`if let Some` 不匹配） | ✅ 原有 `password_hash/salt/iter` 完整保留 |
| — | 调用 `PUT /sends/<id>/remove-password` 独立端点 | ✅ 调用 `set_password(None)`，三个字段全部清空 |

独立的 `put_remove_password` 端点 — [sends.rs:686-L702](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L686-L702)：
```rust
#[put("/sends/<send_id>/remove-password")]
async fn put_remove_password(...) {
    send.set_password(None);  // 唯一能清除 Send 密码的方式
    send.save(&conn).await?;
}
```

**密钥轮换场景下的密码行为**：

```
轮换前：Send 设有访问密码 password_hash = Some(...)
    │
    ▼
客户端构造轮换请求时：
    ├── 方案 A：不发送 password 字段 → data.password = None
    │       └── 服务端不调用 set_password → 原密码保留 ✅
    │
    ├── 方案 B：发送 "password": "原密码" → data.password = Some("原密码")
    │       └── 服务端 set_password(Some("原密码")) → 新 salt + 新哈希，密码逻辑上不变
    │
    └── 方案 C：发送 "password": "新密码" → data.password = Some("新密码")
            └── 服务端 set_password(Some("新密码")) → 密码被修改（等价于顺带改了密码）

⚠️ 发送 "password": null 无法清除密码，必须走 remove-password 独立端点
```

##### NumberOrString：max_access_count 的类型兼容

`max_access_count` 在请求体中是 `Option<NumberOrString>`，这是为了兼容不同客户端序列化的差异：有的客户端发数字 `{"maxAccessCount": 5}`，有的发字符串 `{"maxAccessCount": "5"}`。

[NumberOrString](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/util.rs#L645-L674) 的 untagged 反序列化：
```rust
#[derive(Deserialize)]
#[serde(untagged)]
pub enum NumberOrString {
    Number(i64),
    String(String),
}
```

轮换时的处理 — [sends.rs:647-L650](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs#L647-L650)：
```rust
send.max_access_count = match data.max_access_count {
    Some(m) => Some(m.into_i32()?),  // 数字或字符串统一转 i32
    _ => None,                        // 请求不发该字段 → 清除限制（不设最大访问次数）
};
```

##### 轮换中非加密字段的设计意图

这些访问控制字段与密钥加密体系完全解耦，因此：
- **访问密码（password_hash/salt/iter）**：基于 PBKDF2 本地验证，与 User Key / Send Key 毫无关系，轮换时无需变化
- **max_access_count / access_count**：纯业务计数，与加密无关
- **expiration_date / deletion_date**：纯时间判断，与加密无关
- **hide_email / disabled**：纯布尔开关，与加密无关

所以在密钥轮换中这些字段的"被重新提交"实际上更像是一次全量同步——客户端必须把所有字段都带上，服务端会按语义逐一覆盖（或有条件地保留）。

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
| Send.set_password（密码设置/清除逻辑） | [send.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/send.rs) | L99-L113 |
| Send 访问密码的条件保留判断（if let Some） | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L655-L658 |
| put_remove_password 独立端点（唯一清除密码方式） | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L686-L702 |
| update_send_from_data 限制字段处理 | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L643-L654 |
| NumberOrString 类型兼容定义 | [util.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/util.rs) | L645-L674 |
| SendFileData 文件元数据结构 | [sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/sends.rs) | L365-L371 |
| EmergencyAccess 模型（含 key_encrypted 字段） | [emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs) | L19-L32 |
| EmergencyAccessStatus 状态枚举 | [emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs) | L133-L139 |
| EmergencyAccess.save() 仅更新 grantor 修订版 | [emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs) | L144-L146 |
| EmergencyAccess.find_all_confirmed_by_grantor_uuid | [emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/emergency_access.rs) | L363-L372 |
| 轮换中 Emergency Access 更新循环 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L848-L858 |
| Membership 模型（含 reset_password_key 字段） | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs) | L35-L61 |
| MembershipStatus 状态枚举 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs) | L74-L80 |
| Membership.save() 仅更新当前用户修订版 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs) | L739-L741 |
| Membership.find_by_user（不过滤 status） | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/organization.rs) | L1014-L1022 |
| 轮换中 Organization Reset Password 更新循环 | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L860-L870 |
| 轮换中 membership.retain 过滤（仅保留有 reset_password_key 的） | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/api/core/accounts.rs) | L819-L821 |
| User::update_uuid_revision（全局修订更新函数） | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/129-vaultwarden/src/db/models/user.rs) | L351-L355 |
