# Vaultwarden Bitwarden API 兼容层分析

本文档从代码实现角度梳理 Vaultwarden 与 Bitwarden API 兼容层的设计与实现，重点关注：响应对象如何组织、字段容错如何影响客户端、错误格式细节、以及客户端假设的取舍。

---

## 1. 响应对象组织：`object` 与 `Object` 的适用边界

### 1.1 三类响应格式的明确区分

Vaultwarden API 兼容层存在三套互不兼容的响应鉴别器体系，分别对应三类端点：

| 类别 | 端点示例 | 鉴别器键名 | 值的风格 | 来源模块 |
|---|---|---|---|---|
| **普通 API 响应** | `/api/*` | `"object"`（小写 o） | camelCase 或 PascalCase | `db/models/*` 和 `api/core/*` |
| **Token 响应** | `/identity/connect/token` | `"Object"`（大写 O） | camelCase | `api/identity.rs` 内部嵌套对象 |
| **特殊错误响应** | `/identity/connect/token` 2FA 失败 | 无统一鉴别器 | OAuth2 风格 | `api/identity.rs` |

---

### 1.2 普通 API 响应：`"object"` 小写鉴别器

所有 `/api/*` 端点的响应使用**小写 `"object"`** 作为类型鉴别器，客户端据此分派反序列化逻辑。

**`"object"` 鉴别器值一览表：**

| object 值 | 模型 / 用途 | 代码位置 |
|---|---|---|
| `"profile"` | 用户配置 | `db/models/user.rs:291` |
| `"organization"` | 组织 | `db/models/organization.rs:244` |
| `"profileOrganization"` | 用户-组织成员关系 | `db/models/organization.rs:531` |
| `"organizationUserUserDetails"` | 成员详情 | `db/models/organization.rs:664` |
| `"organizationUserDetails"` | 成员简详情 | `db/models/organization.rs:713` |
| `"organizationUserUserMiniDetails"` | 成员迷你详情 | `db/models/organization.rs:735` |
| `"collection"` | 集合（基础） | `db/models/collection.rs:74` |
| `"collectionDetails"` | 集合（详情） | `db/models/collection.rs:142` |
| `"cipherDetails"` | 密码条目详情 | `db/models/cipher.rs:337` |
| `"attachment"` | 附件 | `db/models/attachment.rs:76` |
| `"folder"` | 文件夹 | `db/models/folder.rs:59` |
| `"policy"` | 组织策略 | `db/models/org_policy.rs:94` |
| `"group"` | 组（基础） | `db/models/group.rs:82` |
| `"groupDetails"` | 组（详情） | `db/models/group.rs:110` |
| `"device"` | 设备 | `db/models/device.rs:72, 129` |
| `"send"` | Send 分享 | `db/models/send.rs:169` |
| `"send-access"` | Send 访问记录 | `db/models/send.rs:191` |
| `"twoFactorProvider"` | 双因素提供者 | `db/models/two_factor.rs:75` |
| `"emergencyAccess"` | 紧急访问 | `db/models/emergency_access.rs:69` |
| `"emergencyAccessGrantorDetails"` | 紧急访问授权方详情 | `db/models/emergency_access.rs:85` |
| `"emergencyAccessGranteeDetails"` | 紧急访问被授权方详情 | `db/models/emergency_access.rs:112` |
| `"list"` | 列表容器 | `api/core/ciphers.rs:220` 等多处 |
| `"sync"` | 同步响应 | `api/core/ciphers.rs:202` |
| `"domains"` | 等效域名 | `api/core/mod.rs:101` |
| `"config"` | 服务器配置 | `api/core/mod.rs:254` |
| `"twoFactorU2f"` | U2F 双因素 | `api/core/two_factor/webauthn.rs:302` 等 |
| `"twoFactorWebAuthn"` | WebAuthn 双因素 | `api/core/two_factor/webauthn.rs:127` |
| `"twoFactorEmail"` | Email 双因素 | `api/core/two_factor/email.rs:145` 等 |
| `"twoFactorDuo"` | Duo 双因素 | `api/core/two_factor/duo.rs:114` 等 |
| `"twoFactorAuthenticator"` | TOTP 认证器 | `api/core/two_factor/authenticator.rs:43` 等 |
| `"twoFactorRecover"` | 双因素恢复 | `api/core/two_factor/mod.rs:117` |
| `"deviceVerificationSettings"` | 设备验证设置 | `api/core/two_factor/mod.rs:300` |
| `"send-fileUpload"` | Send 文件上传 | `api/core/sends.rs:359` |
| `"send-fileDownload"` | Send 文件下载 | `api/core/sends.rs:564` |
| `"organizationKeys"` | 组织密钥 | `api/core/organizations.rs:994` |
| `"organizationPublicKey"` | 组织公钥 | `api/core/organizations.rs:2896` |
| `"apiKey"` | API 密钥 | `api/core/organizations.rs:3146` |
| `"plan"` | 计划信息 | `api/core/organizations.rs:2170, 2179` |
| `"optionalCipherDetails"` | 可选密码条目详情 | `api/core/ciphers.rs:778` |
| `"attachment-fileUpload"` | 附件上传 | `api/core/ciphers.rs:1158` |
| `"register"` | 注册结果 | `api/core/accounts.rs:343` |
| `"emergencyAccessView"` | 紧急访问查看 | `api/core/emergency_access.rs:589` |
| `"emergencyAccessTakeover"` | 紧急访问接管 | `api/core/emergency_access.rs:618` |
| `"error"` | 标准错误 | `error.rs:244, 266` |

**核心模式**：数据库模型手动实现 `to_json()` 方法，通过 `json!()` 宏构建 `serde_json::Value`，所有响应都带 `"object"` 字段。

---

### 1.3 Token 响应：`"Object"` 大写鉴别器

`/identity/connect/token` 端点的**嵌套对象内部**使用**大写 `"Object"`** 作为类型鉴别器。这是 Bitwarden Identity 模块（基于 OAuth2 规范）独立演化的结果。

**`"Object"` 鉴别器值一览表：**

| Object 值 | 用途 | 代码位置 |
|---|---|---|
| `"privateKeys"` | AccountKeys 容器类型 | `api/identity.rs:530, 681` |
| `"publicKeyEncryptionKeyPair"` | 公私钥对类型 | `api/identity.rs:528, 679` |
| `"userDecryptionOptions"` | 用户解密选项 | `api/identity.rs:554, 707` |
| `"masterPasswordPolicy"` | 主密码策略 | `api/mod.rs:124`（生成），`api/identity.rs:548`（Token），`api/identity.rs:912`（2FA） |

**Token 响应的完整 PascalCase 风格：**

```rust
// api/identity.rs:502 — 调用 master_password_policy() 函数生成策略对象
let master_password_policy = master_password_policy(user, conn).await;

let mut result = json!({
    "access_token": auth_tokens.access_token(),  // OAuth2 标准字段（下划线）
    "expires_in": auth_tokens.expires_in(),      // OAuth2 标准字段
    "token_type": "Bearer",                      // OAuth2 标准字段
    "refresh_token": auth_tokens.refresh_token(),// OAuth2 标准字段
    
    // 以下全部 PascalCase
    "PrivateKey": user.private_key,
    "Kdf": user.client_kdf_type,
    "KdfIterations": user.client_kdf_iter,
    "KdfMemory": user.client_kdf_memory,
    "KdfParallelism": user.client_kdf_parallelism,
    "ResetMasterPassword": false,
    "ForcePasswordReset": false,
    "MasterPasswordPolicy": master_password_policy,  // ← 函数返回值，含大写 Object
    "scope": auth_tokens.scope(),
    "AccountKeys": {
        "publicKeyEncryptionKeyPair": {
            "wrappedPrivateKey": user.private_key,
            "publicKey": user.public_key,
            "Object": "publicKeyEncryptionKeyPair"  // 大写 Object
        },
        "Object": "privateKeys"  // 大写 Object
    },
    "UserDecryptionOptions": {
        "HasMasterPassword": has_master_password,
        "MasterPasswordUnlock": master_password_unlock,
        "Object": "userDecryptionOptions"  // 大写 Object
    },
});
```
— `api/identity.rs:536-556`

**关键设计取舍**：OAuth2 标准字段使用下划线（`access_token`, `expires_in`, `token_type`, `refresh_token`），Bitwarden 扩展字段全部使用 PascalCase，这是 OAuth2 规范与 Bitwarden 私有扩展的分界线。

---

### 1.4 `MasterPasswordPolicy` 的三处用法与大小写对照

`MasterPasswordPolicy` 是整个兼容层中大小写问题最复杂的对象，它在三处出现，每处的数据来源和字段大小写都不同：

#### 来源：`master_password_policy()` 函数（`api/mod.rs:93-126`）

```rust
#[derive(Debug, Default, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]  // ← 序列化字段为 camelCase
pub struct MasterPasswordPolicy {
    min_complexity: Option<u8>,    // → 序列化为 minComplexity
    min_length: Option<u32>,       // → 序列化为 minLength
    require_lower: bool,           // → 序列化为 requireLower
    require_upper: bool,           // → 序列化为 requireUpper
    require_numbers: bool,         // → 序列化为 requireNumbers
    require_special: bool,         // → 序列化为 requireSpecial
    enforce_on_login: bool,        // → 序列化为 enforceOnLogin
}

async fn master_password_policy(user: &User, conn: &DbConn) -> Value {
    // ... 合并策略 ...
    let mut mpp_json = if !master_password_policies.is_empty() {
        json!(reduced_policy)  // Serde 序列化 → camelCase 字段
    } else if CONFIG.sso_enabled() {
        CONFIG.sso_master_password_policy_value().unwrap_or(json!({}))
    } else {
        json!({})
    };
    // NOTE: Upstream still uses PascalCase here for `Object`!
    mpp_json["Object"] = json!("masterPasswordPolicy");  // ← 手动注入大写 Object
    mpp_json
}
```

**关键**：结构体通过 `#[serde(rename_all = "camelCase")]` 序列化为 camelCase 字段名，但 `Object` 键是手动注入的，不受 Serde 控制。最终输出的 JSON 内部字段是 camelCase，但鉴别器是大写 `Object`。

#### 用法一：Token 响应（`api/identity.rs:548`）

```rust
"MasterPasswordPolicy": master_password_policy,  // 调用函数的返回值
```

输出示例（有策略时）：
```json
{
    "MasterPasswordPolicy": {         // PascalCase 外壳键名
        "minComplexity": 3,           // camelCase（Serde 序列化）
        "minLength": 12,              // camelCase
        "requireLower": false,        // camelCase
        "requireUpper": false,        // camelCase
        "requireNumbers": false,      // camelCase
        "requireSpecial": false,      // camelCase
        "enforceOnLogin": false,      // camelCase
        "Object": "masterPasswordPolicy"  // 大写 Object（手动注入）
    }
}
```

输出示例（无策略时）：
```json
{
    "MasterPasswordPolicy": {         // PascalCase 外壳键名
        "Object": "masterPasswordPolicy"  // 大写 Object，无策略字段
    }
}
```

#### 用法二：2FA 错误响应（`api/identity.rs:911-912`）

```rust
"MasterPasswordPolicy": {
    "Object": "masterPasswordPolicy"
}
```

**硬编码空对象**，不调用 `master_password_policy()` 函数。2FA 验证流程此时尚未完成登录，不提供策略细节，仅提供鉴别器让客户端能正确解析。

输出：
```json
{
    "error": "invalid_grant",
    "error_description": "Two factor required.",
    "TwoFactorProviders": ["0", "1"],
    "TwoFactorProviders2": { "0": null, "1": null },
    "MasterPasswordPolicy": {              // PascalCase 外壳键名
        "Object": "masterPasswordPolicy"   // 大写 Object，硬编码空策略
    }
}
```

#### 用法三：`MasterPasswordPolicy` 不出现在 sync 响应中

`/api/sync` 端点不返回 `MasterPasswordPolicy`。密码策略通过 `policies` 数组（`OrgPolicy::to_json()`）传递，该数组内每个策略对象使用小写 `"object": "policy"` 鉴别器（见 1.2 节），格式与 Token 响应完全不同。

#### 三处用法大小写对照表

| 维度 | Token 响应 | 2FA 错误 | sync 策略数组 |
|---|---|---|---|
| **外壳键名** | `"MasterPasswordPolicy"` (PascalCase) | `"MasterPasswordPolicy"` (PascalCase) | N/A（在 policies 数组内） |
| **数据来源** | `master_password_policy()` 函数 | 硬编码空对象 | `OrgPolicy::to_json()` |
| **内部字段大小写** | camelCase（Serde 序列化） | 无内部字段 | camelCase（`#[serde(rename_all)]`） |
| **鉴别器键名** | `"Object"`（大写 O） | `"Object"`（大写 O） | `"object"`（小写 o） |
| **鉴别器值** | `"masterPasswordPolicy"` | `"masterPasswordPolicy"` | `"policy"` |
| **代码位置** | `api/identity.rs:548` | `api/identity.rs:911-912` | `db/models/org_policy.rs:94` |

**核心规律**：同一数据在不同响应格式中使用不同大小写的鉴别器——Token/2FA 错误的大写 `"Object"` 是 Identity 模块（OAuth2 上下文）的约定；sync 的策略数组使用小写 `"object"` 是普通 API 响应的约定。`masterPasswordPolicy` 这个鉴别器**值**本身（不是键名）始终是 camelCase 首字母小写，即使在 PascalCase 上下文中也是如此。

---

### 1.5 sync 端点的特殊混合：`"object"` + camelCase

`/api/sync` 属于普通 API 响应（`"object": "sync"` 小写），但其内部嵌套的 `userDecryption` 结构与 Token 响应的 `UserDecryptionOptions` 语义相同但**大小写完全不同**：

```rust
// /api/sync 响应
json!({
    "profile": user_json,
    "folders": folders_json,
    "collections": collections_json,
    "policies": policies_json,
    "ciphers": ciphers_json,
    "domains": domains_json,
    "sends": sends_json,
    "userDecryption": {           // camelCase（对比 PascalCase 的 UserDecryptionOptions）
        "masterPasswordUnlock": {
            "kdf": {              // 全小写（对比 PascalCase Kdf/KdfType）
                "kdfType": headers.user.client_kdf_type,
                "iterations": headers.user.client_kdf_iter,
                "memory": headers.user.client_kdf_memory,
                "parallelism": headers.user.client_kdf_parallelism
            },
            "masterKeyEncryptedUserKey": headers.user.akey,  // 全小写（对比 PascalCase）
            "masterKeyWrappedUserKey": headers.user.akey,
            "salt": headers.user.email
        },
    },
    "object": "sync"  // 小写 object
})
```
— `api/core/ciphers.rs:191-203`

源码注释明确指出了这个不一致：

```rust
// This is very similar to the the userDecryptionOptions sent in connect/token,
// but as of 2025-12-19 they're both using different casing conventions.
```
— `api/core/ciphers.rs:170-172`

---

### 1.6 错误响应的 `object` 使用

标准错误响应（ApiErrorResponse 和 CompactApiErrorResponse）也使用**小写 `"object": "error"`**：

```rust
// ApiErrorResponse（9 字段）
state.serialize_field("object", "error")?;  // 小写 object

// CompactApiErrorResponse（6 字段）
state.serialize_field("object", "error")?;  // 小写 object
```
— `error.rs:244, 266`

但 2FA 特殊错误响应（OAuth2 风格）**不带 `"object"` 鉴别器**，使用 `error` + `error_description` 字段。

---

### 1.7 同一模型的多级响应

Bitwarden 上游对同一实体定义了信息密度递增的多级响应模型：

- **Cipher**：`cipherMini` → `cipher` → `cipherDetails`
- **Collection**：`collection` → `collectionDetails`
- **Group**：`group` → `groupDetails`
- **OrganizationUser**：`organizationUserDetails` → `organizationUserUserDetails` → `organizationUserUserMiniDetails`

**Vaultwarden 的取舍**：
- Cipher：**始终返回最详细的 `cipherDetails`**，依赖客户端忽略多余字段
- Collection：同时提供 `collection`（基础版）和 `collection_details`（详情版）两个方法
- Group：同时提供 `to_json`（基础版）和 `to_json_details`（详情版）两个方法

```rust
// There are three types of cipher response models in upstream
// Bitwarden: "cipherMini", "cipher", and "cipherDetails" (in order
// of increasing level of detail). vaultwarden currently only
// supports the "cipherDetails" type, though it seems like the
// Bitwarden clients will ignore extra fields.
```
— `db/models/cipher.rs:329-335`

---

### 1.8 Cipher 响应的双层结构

Cipher 响应同时包含顶层字段和类型特定子对象，存在数据冗余设计：

**顶层字段**（所有 cipher 类型共享）：

```rust
json!({
    "object": "cipherDetails",
    "id": self.uuid,
    "type": self.atype,           // 1=Login, 2=SecureNote, 3=Card, 4=Identity, 5=SshKey
    "creationDate": format_date(&self.created_at),
    "revisionDate": format_date(&self.updated_at),
    "deletedDate": ...,
    "reprompt": ...,
    "organizationId": self.organization_uuid,
    "key": self.key,
    "attachments": attachments_json,
    "organizationUseTotp": true,  // 配合 usersGetPremium 控制 TOTP 计数器显示
    "collectionIds": collection_ids,
    "name": self.name,
    "notes": self.notes,
    "fields": fields_json,
    "data": data_json,            // ← 与类型子对象内容重叠
    "passwordHistory": password_history_json,
    "login": null, "secureNote": null, "card": null, "identity": null, "sshKey": null,
})
```
— `db/models/cipher.rs:336-368`

**类型子对象**：根据 `atype` 值，将 `type_data_json` 填入对应 key（login/secureNote/card/identity/sshKey），其他保持 `null`。

**`data` 与类型子对象的冗余**：`data_json` 是 `type_data_json` 的克隆再追加了 fields/name/notes/passwordHistory，而类型子对象则是原始的 `type_data_json`。这意味着 `data.login.fields` 和顶层 `fields` 存在数据冗余——这是上游的遗留设计。

**sync_type 控制的条件字段**：`CipherSyncType` 分为 `User` 和 `Organization` 两种。仅在 `User` 同步时追加 folderId/favorite/archivedDate/edit/viewPassword/permissions 字段，组织同步时省略。

---

### 1.9 列表响应格式

列表端点（如 `GET /api/ciphers`）使用 `"object": "list"` + `"data"` 数组 + `"continuationToken"` 的格式：

```rust
Ok(Json(json!({
    "data": ciphers_json,
    "object": "list",
    "continuationToken": null
})))
```
— `api/core/ciphers.rs:218-222`

---

### 1.10 Membership 的 Manager → Custom 类型映射 HACK

Bitwarden 有 Owner(0)/Admin(1)/User(2)/Manager(3)/Custom(4) 五种成员类型。Vaultwarden 将 Manager(3) 在输出时映射为 Custom(4)，因为需要利用 Custom 类型的 permissions 对象来模拟 Manager 的集合权限。这是一个有意的 HACK：

```rust
/// HACK: Convert the manager type to a custom type
/// It will be converted back on other locations
pub fn type_manager_as_custom(&self) -> i32 {
    match self.atype {
        3 => 4,
        _ => self.atype,
    }
}
```
— `db/models/organization.rs:310-317`

同时在反序列化时，`"4" | "Custom"` 也会映射回 `MembershipType::Manager`。

---

## 2. 字段容错与客户端影响

### 2.1 `LowerCase<T>`：字段名首字母小写归一化

`LowerCase<T>` 不是简单的"大小写不敏感"，而是**将 JSON 对象的所有键名首字母转为小写**后递归处理。这是为了应对 Bitwarden 客户端在不同版本/平台发送的键名大小写不一致问题。

```rust
pub struct LowerCase<T: DeserializeOwned> {
    #[serde(deserialize_with = "lowercase_deserialize")]
    #[serde(flatten)]
    pub data: T,
}
```
— `util.rs:560-565`

核心逻辑：

```rust
pub fn lcase_first(s: &str) -> String {
    let mut c = s.chars();
    match c.next() {
        None => String::new(),
        Some(f) => f.to_lowercase().collect::<String>() + c.as_str(),
    }
}
```
— `util.rs:373-379`

**特殊情况处理**：对 `ssn`（社保号）键做了硬编码特殊处理，因为 `lcase_first("SSN")` 会变成 `sSN`，需要保持为 `ssn`。

**影响范围**：Cipher 的 `fields`、`password_history`、`data` 列在 DB 中存储的 JSON，反序列化时都经过 `LowerCase<Value>` 处理，确保无论客户端发来 `Uri` 还是 `uri`、`Match` 还是 `match`，都能正确识别。

---

### 2.2 `NumberOrString`：数值/字符串双态

某些字段（如 device_type）客户端可能发送数字或字符串，`NumberOrString` 枚举处理这种双态：

```rust
#[derive(Clone, Debug, Deserialize)]
#[serde(untagged)]
pub enum NumberOrString {
    Number(i64),
    String(String),
}
```
— `util.rs:645-650`

---

### 2.3 fields.type 必须为数字

**客户端崩溃风险**：fields 中的 `type` 键如果不是数字而是字符串，会导致某些客户端（尤其是移动端）崩溃。Vaultwarden 在 `to_json()` 中强制修正，回退到隐藏类型 `1` 以防止意外数据泄露：

```rust
match f.data.get("type") {
    Some(t) if t.is_number() => {}
    Some(t) if t.is_string() => {
        let type_num = &t.as_str().unwrap_or("1").parse::<u8>().unwrap_or(1);
        f.data["type"] = json!(type_num);
    }
    _ => {
        f.data["type"] = json!(1); // 回退为隐藏类型，防止数据泄露
    }
}
```
— `db/models/cipher.rs:205-214`

---

### 2.4 uri.match 必须为数字或 null

同理，Login 类型的 `uris[].match` 字段如果为字符串，会导致客户端异常，需要转为数字或 null：

```rust
for uri in &mut *uris {
    if uri["match"].is_string() {
        let match_value = match uri["match"].as_str().unwrap_or_default().parse::<u8>() {
            Ok(n) => json!(n),
            _ => Value::Null,
        };
        uri["match"] = match_value;
    }
}
```
— `db/models/cipher.rs:267-275`

---

### 2.5 passwordHistory 中 password 为 null 的条目必须过滤

**客户端崩溃风险**：较新版本的 Bitwarden 客户端无法处理 passwordHistory 中 `password` 字段为 `null` 的条目，Vaultwarden 直接过滤掉：

```rust
.filter_map(|d| match d.data.get("password") {
    Some(p) if p.is_string() => Some(d.data),
    _ => None, // null password 的条目被丢弃
})
```
— `db/models/cipher.rs:233-237`

---

### 2.6 type_data 解析失败时必须返回空对象

**移动端崩溃风险**：如果 cipher 的 `data` 字段无法解析为 JSON 对象，不能返回 null，必须回退为空对象 `{}`：

```rust
let mut type_data_json = serde_json::from_str::<LowerCase<Value>>(&self.data)
    .inspect_err(|_| warn!("Error parsing data field for {}", self.uuid))
    .map_or_else(|_| Value::Object(serde_json::Map::new()), |d| d.data);
```
— `db/models/cipher.rs:251-255`

---

### 2.7 SecureNote 的 type 字段必须为数字

**原生移动端崩溃风险**：SecureNote（atype=2）的 `type_data_json` 必须包含数字类型的 `type` 键，否则重置为 `{"type": 0}`：

```rust
if self.atype == 2 {
    match type_data_json {
        Value::Object(ref t) if t.get("type").is_some_and(Value::is_number) => {}
        _ => {
            type_data_json = json!({"type": 0});
        }
    }
}
```
— `db/models/cipher.rs:285-294`

---

### 2.8 SSH Key 缺失必填字段时设为 null

**移动端崩溃风险**：SSH Key（atype=5）如果 keyFingerprint/privateKey/publicKey 任一缺失或为空，唯一安全的处理方式是将整个 type_data 设为 null。源码注释承认这仍可能导致打开该条目时客户端崩溃，但至少允许编辑/保存/删除：

```rust
if self.atype == 5
    && (type_data_json["keyFingerprint"].as_str().is_none_or(str::is_empty)
        || type_data_json["privateKey"].as_str().is_none_or(str::is_empty)
        || type_data_json["publicKey"].as_str().is_none_or(str::is_empty))
{
    warn!("Error parsing ssh-key, mandatory fields are invalid for {}", self.uuid);
    type_data_json = Value::Null;
}
```
— `db/models/cipher.rs:296-307`

---

### 2.9 Login 类型的 uri 向后兼容

**多个（移动端）客户端依赖**：尽管标记为"向后兼容代码"，但截至 2021 年 1 月上游仍在使用。将 `uris` 数组首个元素的 `uri` 值复制到顶层 `uri` 字段：

```rust
if self.atype == 1 {
    type_data_json["uri"] = Value::Null; // 始终存在，默认 null
    if let Some(uris) = type_data_json["uris"].as_array_mut()
        && !uris.is_empty()
    {
        type_data_json["uri"] = uris[0]["uri"].clone();
    }
}
```
— `db/models/cipher.rs:257-277`

---

### 2.10 data_json 的必含字段

无论 cipher 类型，`data_json`（即 `type_data_json` 的克隆）始终注入四个字段。这同样是因为某些客户端假设这些字段总在 data 内：

```rust
data_json["fields"] = json!(fields_json);
data_json["name"] = json!(self.name);
data_json["notes"] = json!(self.notes);
data_json["passwordHistory"] = Value::Array(password_history_json.clone());
```
— `db/models/cipher.rs:312-317`

---

## 3. 错误响应

### 3.1 四套错误格式汇总

| 格式名称 | 使用场景 | 特征字段 | 代码位置 |
|---|---|---|---|
| **ApiErrorResponse**（9 字段） | 大多数 API 错误 | `validationErrors: {"": ["msg"]}`, `errorModel`, `error: ""`, `error_description: ""` | `error.rs:214-248` |
| **CompactApiErrorResponse**（6 字段） | 部分较新错误 | `validationErrors: null`，无 errorModel/error/error_description | `error.rs:254-270` |
| **OAuth2 风格错误** | 2FA 要求、refresh_token 失效 | `error: "invalid_grant"`, `error_description`, `TwoFactorProviders2` | `api/identity.rs:138-160, 906-914` |
| **404 Catcher 错误** | 路由不存在 | `{"error": {"code": 404, "reason": "...", "description": "..."}}` | `api/core/mod.rs:262-271` |

---

### 3.2 ApiErrorResponse（9 字段）

用于大多数 API 错误，通过自定义 `Serialize` 实现：

```rust
impl Serialize for ApiErrorResponse<'_> {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer,
    {
        let mut state = serializer.serialize_struct("ApiErrorResponse", 9)?;
        state.serialize_field("message", self.0.message)?;

        let mut validation_errors = std::collections::HashMap::with_capacity(1);
        validation_errors.insert("", vec![self.0.message]);
        state.serialize_field("validationErrors", &validation_errors)?;

        let error_model = ErrorModel { message: self.0.message, object: "error" };
        state.serialize_field("errorModel", &error_model)?;

        state.serialize_field("error", "")?;
        state.serialize_field("error_description", "")?;
        state.serialize_field("exceptionMessage", &None::<()>)?;
        state.serialize_field("exceptionStackTrace", &None::<()>)?;
        state.serialize_field("innerExceptionMessage", &None::<()>)?;
        state.serialize_field("object", "error")?;
        state.end()
    }
}
```
— `error.rs:214-248`

---

### 3.3 CompactApiErrorResponse（6 字段）

用于部分较新的错误响应，比 ApiErrorResponse 少 3 个字段：

```rust
impl Serialize for CompactApiErrorResponse<'_> {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer,
    {
        let mut state = serializer.serialize_struct("CompactApiErrorResponse", 6)?;
        state.serialize_field("message", self.0.message)?;
        state.serialize_field("validationErrors", &None::<()>)?;
        state.serialize_field("exceptionMessage", &None::<()>)?;
        state.serialize_field("exceptionStackTrace", &None::<()>)?;
        state.serialize_field("innerExceptionMessage", &None::<()>)?;
        state.serialize_field("object", "error")?;
        state.end()
    }
}
```
— `error.rs:254-270`

**两者的关键差异**：

| 字段 | ApiErrorResponse | CompactApiErrorResponse |
|---|---|---|
| `message` | ✅ | ✅ |
| `validationErrors` | `{"": ["msg"]}` | `null` |
| `errorModel` | ✅ (含 message+object) | ❌ |
| `error` | `""` | ❌ |
| `error_description` | `""` | ❌ |
| `exceptionMessage` | `null` | `null` |
| `exceptionStackTrace` | `null` | `null` |
| `innerExceptionMessage` | `null` | `null` |
| `object` | `"error"` | `"error"` |

---

### 3.4 错误路由机制：Display trait 选择序列化函数

`make_error!` 宏为每种 `ErrorKind` 绑定了一个"用户消息生成函数"，这个函数决定了最终的 JSON 格式：

- `serialize` → 直接序列化内部类型（用于 Empty 和 Json）
- `api_error` → 使用 `ApiErrorResponse` 包装
- `compact_api_error` → 使用 `CompactApiErrorResponse` 包装

```rust
make_error! {
    Empty(Empty):     no_source, serialize,          // → 空对象 {}
    Simple(String):  no_source,  api_error,          // → ApiErrorResponse
    Compact(Compact):  no_source,  compact_api_error, // → CompactApiErrorResponse
    Json(Value):      no_source,  serialize,          // → 自定义 JSON
    Db(DieselErr):    has_source, api_error,          // → ApiErrorResponse
    // ... 所有其他类型 → api_error
}
```
— `error.rs:73-108`

**关键点**：`Error::to_string()` 的输出就是 HTTP 响应 body。`Display` trait 实现调用对应的生成函数，这个函数返回序列化后的 JSON 字符串。然后 Rocket 的 `Responder` 实现将其包装为 JSON 响应。

---

### 3.5 OAuth2 风格错误（2FA 要求）

2FA 错误使用 `err_json!` 绕过标准错误格式，发送 OAuth2 风格的响应。注意 `MasterPasswordPolicy` 是**硬编码空对象**，不调用 `master_password_policy()` 函数（见 1.4 节用法二）：

```rust
json!({
    "error" : "invalid_grant",
    "error_description" : "Two factor required.",
    "TwoFactorProviders" : [...],
    "TwoFactorProviders2" : { ... },
    "MasterPasswordPolicy": {
        "Object": "masterPasswordPolicy"  // 硬编码空策略，非函数返回值
    }
})
```
— `api/identity.rs:906-914`

客户端在收到 `error: "invalid_grant"` 时会检查 `TwoFactorProviders2` 并触发 2FA 流程。如果这里使用标准 ApiErrorResponse 格式，客户端将无法识别 2FA 要求。

---

### 3.6 OAuth2 风格错误（refresh_token 失效）

refresh_token 失效时同样需要 `{"error": "invalid_grant"}` 格式，客户端据此强制登出：

```rust
err_json!(json!({"error": "invalid_grant"}), "Missing refresh_token")
```
— `api/identity.rs:144-146`

---

### 3.7 自定义验证错误（Cipher 批量导入）

使用 `err_json!` 发送带字段级 `validationErrors` 的响应：

```rust
let err_json = json!({
    "message": "The model state is invalid.",
    "validationErrors" : validation_errors, // 如 {"Ciphers[0].Notes": ["..."]}
    "object": "error"
});
err_json!(err_json, "Import validation errors")
```
— `db/models/cipher.rs:131-137`

---

### 3.8 404 Catcher

Rocket 的 404 catcher 使用与标准错误完全不同的格式：

```rust
#[catch(404)]
fn api_not_found() -> Json<Value> {
    Json(json!({
        "error": {
            "code": 404,
            "reason": "Not Found",
            "description": "The requested resource could not be found."
        }
    }))
}
```
— `api/core/mod.rs:262-271`

这是第四种错误格式。客户端对 404 的处理路径与业务错误不同。

---

## 4. 客户端假设与取舍

### 4.1 config 端点的 version 字段

```rust
"version": "2025.12.0",
```
— `api/core/mod.rs:231`

```rust
// Note: The clients use this version to handle backwards compatibility concerns
// This means they expect a version that closely matches the Bitwarden server version
// We should make sure that we keep this updated when we support the new server features
// Version history:
// - Individual cipher key encryption: 2024.2.0
// - Mobile app support for MasterPasswordUnlockData: 2025.8.0
```

**取舍**：Vaultwarden 必须声明一个与 Bitwarden 服务器版本兼容的版本号，否则客户端会因版本检查而禁用功能或使用错误的代码路径。

---

### 4.2 SSH Key 按客户端版本过滤

```rust
let show_ssh_keys = if let Some(client_version) = client_version {
    let ver_match = semver::VersionReq::parse(">=2024.12.0").unwrap();
    ver_match.matches(&client_version.0)
} else {
    false  // 无法获取版本时默认不显示
};
if !show_ssh_keys {
    ciphers.retain(|c| c.atype != 5);
}
```
— `api/core/ciphers.rs:128-137`

**取舍**：如果客户端版本低于 2024.12.0，完全移除 SSH Key 类型的 cipher。无法获取版本信息时也默认不显示——宁可丢失数据也不让旧客户端崩溃。

---

### 4.3 2FA Email 发送行为的版本控制

```rust
// Starting with version 2025.5.0 the client will call `/api/two-factor/send-email-login`.
let disabled_send = if let Some(cv) = client_version {
    let ver_match = semver::VersionReq::parse(">=2025.5.0").unwrap();
    ver_match.matches(&cv.0)
} else {
    false
};

// Send email immediately if email is the only 2FA option.
if providers.len() == 1 && !disabled_send {
    email::send_token(user_id, conn).await?;
}
```
— `api/identity.rs:972-983`

**取舍**：2025.5.0+ 的客户端会自己调用专门的 email 发送端点，服务端不应重复发送。

---

### 4.4 webauthn 端点空响应

```rust
#[get("/webauthn")]
fn get_api_webauthn(_headers: Headers) -> Json<Value> {
    // Prevent a 404 error, which also causes key-rotation issues
    Json(json!({
        "object": "list",
        "data": [],
        "continuationToken": null
    }))
}
```
— `api/core/mod.rs:199-208`

**取舍**：Vaultwarden 暂不支持 passkey 登录，但必须提供此端点。返回 404 会导致客户端密钥轮换问题，返回空列表则安全通过。

---

### 4.5 MasterPasswordUnlockData 的字段名过渡

```rust
// This field is named inconsistently and will be removed and replaced by the "wrapped" variant in the apps.
// https://github.com/bitwarden/android/blob/release/2025.12-rc41/network/src/main/kotlin/com/bitwarden/network/model/MasterPasswordUnlockDataJson.kt#L22-L26
"MasterKeyEncryptedUserKey": user.akey,
"MasterKeyWrappedUserKey": user.akey,
```
— `api/identity.rs:664-668`

**取舍**：同时输出新旧两个键名（值相同），确保新旧客户端都能正确解密。旧键名 `MasterKeyEncryptedUserKey` 计划在未来被移除。

---

### 4.6 Premium 功能全部解锁

```rust
// User::to_json()
"premium": true,
"premiumFromOrganization": false,

// Organization::to_json()
"usersGetPremium": true,

// Membership::to_json()
"usersGetPremium": true,
```

**取舍**：Vaultwarden 作为自托管服务没有付费层，所以将所有 Premium 功能标记为已启用。客户端据此启用 TOTP、附件、字段等高级功能。

---

### 4.7 组织功能标志的硬编码

Organization::to_json() 和 Membership::to_json() 中大量功能标志硬编码为 false，对应 Vaultwarden 不支持的功能：

| 标志 | 值 | 原因 |
|---|---|---|
| `useScim` | false | 非 AGPLv3 许可 |
| `useSecretsManager` | false | 非 AGPLv3 许可 |
| `useKeyConnector` | false | 不支持 |
| `useSso` | false | 不支持 |
| `ssoBound` | false | 不支持 |
| `useRiskInsights` | false | 非 AGPLv3 许可 |
| `useAdminSponsoredFamilies` | false | 不支持 |
| `useActivateAutofillPolicy` | false | 不支持 |

动态值的标志：
- `useEvents` → `CONFIG.org_events_enabled()`
- `useGroups` → `CONFIG.org_groups_enabled()`
- `useResetPassword` → `CONFIG.mail_enabled()`

---

### 4.8 存储容量无限制

```rust
"maxStorageGb": i16::MAX, // The value doesn't matter, we don't check server-side
```
— `db/models/organization.rs:205`

**取舍**：设置为 `i16::MAX`（32767 GB）而非 null，因为某些客户端在 null 时可能显示为 0 GB 限制。

---

### 4.9 planType 和 productTierType

```rust
// Organization::to_json()
"planType": 6, // Custom plan

// Membership::to_json()
"productTierType": 3, // Enterprise tier
```

**取舍**：planType=6（Custom）和 productTierType=3（Enterprise）确保客户端不会显示付费相关的 UI 元素（如升级提示）。

---

### 4.10 seats 硬编码

```rust
// Membership::to_json()
"seats": 20, // hardcoded maxEmailsCount in the web-vault
```
— `db/models/organization.rs:474`

Web Vault 使用 `seats` 值作为邀请用户时的最大邮件数上限。

---

### 4.11 limitCollectionCreation 按 MembershipType 控制

```rust
// Membership::to_json()
"limitCollectionCreation": self.atype < MembershipType::Manager || !self.access_all,
```
— `db/models/organization.rs:513`

**取舍**：将集合创建权限限制在 Manager 及以上级别且具有 access_all 权限的用户，防止权限问题。而 Organization::to_json() 中 `limitCollectionCreation` 始终为 true，因为组织级别的设置不同。

---

## 5. 总结

Vaultwarden 的 API 兼容层设计可归纳为以下核心原则：

1. **三套响应鉴别器体系互不兼容**：
   - 普通 API 响应 → `"object"`（小写 o）
   - Token 响应内部嵌套对象 → `"Object"`（大写 O）
   - OAuth2 风格错误 → 无鉴别器，靠 `error: "invalid_grant"` 识别

2. **大小写不一致是常态**：sync 端点的 `userDecryption` 是 camelCase，Token 端点的 `UserDecryptionOptions` 是 PascalCase；`MasterPasswordPolicy` 对象内部字段是 camelCase（Serde 序列化），但鉴别器键名是 `"Object"`（大写 O），而 sync 策略数组中同一数据用 `"object"`（小写 o）。这是 Bitwarden 上游各模块独立演化的历史遗留。

3. **字段容错的核心动机是防止客户端崩溃**：fields.type 为字符串→崩溃；type_data 为 null→崩溃；SecureNote 缺少 type→崩溃；SSH Key 缺必填字段→崩溃。这些修正不是"锦上添花"而是"不做就崩"，说明 Bitwarden 客户端在反序列化路径上缺乏防御性编程。

4. **错误响应有四套互不兼容的格式**：ApiErrorResponse（9 字段）、CompactApiErrorResponse（6 字段）、OAuth2 风格的 2FA/refresh_token 错误、以及 404 Catcher 错误。客户端在四条不同代码路径上分别处理。

5. **客户端假设的取舍倾向于"安全优先"**：无法确定客户端版本时默认不显示 SSH Key；不支持的功能端点返回空数据而非 404；Premium 功能全部解锁以避免功能被错误禁用；版本号必须与上游同步以通过客户端的兼容性检查。
