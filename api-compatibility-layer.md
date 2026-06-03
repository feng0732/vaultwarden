# Vaultwarden Bitwarden API 兼容层分析

本文档从代码实现角度梳理 Vaultwarden 与 Bitwarden API 兼容层的设计与实现，重点关注：响应对象如何组织、字段容错如何影响客户端、错误格式细节、以及客户端假设的取舍。

---

## 1. 响应对象组织

### 1.1 核心机制：`to_json()` 方法族 + `object` 鉴别器

Vaultwarden 没有使用 Serde derive 宏生成 API 响应结构体，而是在每个数据库模型上手动实现 `to_json()` 方法，通过 `json!()` 宏构建 `serde_json::Value`。每个响应都带有一个 `"object"` 字段作为类型鉴别器，客户端据此决定如何反序列化。

**`object` 鉴别器值一览表：**

| object 值 | 模型 | 来源 |
|---|---|---|
| `"profile"` | 用户配置 | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/user.rs#L291) |
| `"organization"` | 组织 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L244) |
| `"profileOrganization"` | 用户-组织成员关系 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L531) |
| `"organizationUserUserDetails"` | 成员详情 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L664) |
| `"organizationUserDetails"` | 成员简详情 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L713) |
| `"collection"` | 集合 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/collection.rs#L74) |
| `"collectionDetails"` | 集合详情 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/collection.rs#L142) |
| `"cipherDetails"` | 密码条目详情 | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L337) |
| `"list"` | 列表容器 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/ciphers.rs#L220) |
| `"sync"` | 同步响应 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/ciphers.rs#L203) |
| `"domains"` | 等效域名 | [mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/mod.rs#L101) |
| `"masterPasswordPolicy"` | 主密码策略 | [mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/mod.rs#L124) |
| `"userDecryptionOptions"` | 用户解密选项 | [identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/identity.rs#L554) |
| `"config"` | 服务器配置 | [mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/mod.rs#L254) |
| `"error"` | 错误 | [error.rs](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/error.rs#L244) |

### 1.2 同一模型的多级响应

Bitwarden 上游对同一实体定义了信息密度递增的多级响应模型（如 `cipherMini` → `cipher` → `cipherDetails`）。Vaultwarden 的取舍是**始终返回最详细的 `cipherDetails`**，源码注释说明了原因：

```rust
// There are three types of cipher response models in upstream
// Bitwarden: "cipherMini", "cipher", and "cipherDetails" (in order
// of increasing level of detail). vaultwarden currently only
// supports the "cipherDetails" type, though it seems like the
// Bitwarden clients will ignore extra fields.
```
— [cipher.rs#L329-L335](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L329-L335)

类似地，Collection 有 `collection` 和 `collectionDetails` 两级：基础版本只有 id/name/organizationId/externalId/object，而 details 版本在此基础上追加 readOnly/hidePasswords/manage，并将 `object` 从 `"collection"` 改写为 `"collectionDetails"`。— [collection.rs#L68-L147](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/collection.rs#L68-L147)

### 1.3 Cipher 响应的双层结构

Cipher 响应是所有模型中最复杂的，它同时包含顶层字段和类型特定子对象：

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
— [cipher.rs#L336-L368](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L336-L368)

**类型子对象**：根据 `atype` 值，将 `type_data_json` 填入对应 key（login/secureNote/card/identity/sshKey），其他保持 `null`。— [cipher.rs#L401-L410](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L401-L410)

**`data` 与类型子对象的冗余**：`data_json` 是 `type_data_json` 的克隆再追加了 fields/name/notes/passwordHistory，而类型子对象则是原始的 `type_data_json`。这意味着 `data.login.fields` 和顶层 `fields` 存在数据冗余——这是上游的遗留设计。

**sync_type 控制的条件字段**：`CipherSyncType` 分为 `User` 和 `Organization` 两种。仅在 `User` 同步时追加以下字段，组织同步时省略：

```rust
if sync_type == CipherSyncType::User {
    json_object["folderId"] = ...;
    json_object["favorite"] = ...;
    json_object["archivedDate"] = ...;
    json_object["edit"] = json!(!read_only);
    json_object["viewPassword"] = json!(!hide_passwords);
    json_object["permissions"] = json!({ "delete": !read_only, "restore": !read_only });
}
```
— [cipher.rs#L373-L399](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L373-L399)

### 1.4 同步响应

`/api/sync` 是客户端数据拉取的核心端点，一次返回所有数据：

```rust
json!({
    "profile": user_json,
    "folders": folders_json,
    "collections": collections_json,
    "policies": policies_json,
    "ciphers": ciphers_json,
    "domains": domains_json,
    "sends": sends_json,
    "userDecryption": { "masterPasswordUnlock": master_password_unlock },
    "object": "sync"
})
```
— [ciphers.rs#L191-L203](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/ciphers.rs#L191-L203)

其中 `userDecryption` 使用 **camelCase**，而同一数据在 `/connect/token` 中以 `UserDecryptionOptions` / **PascalCase** 输出。源码注释明确指出了这个不一致：

```rust
// This is very similar to the the userDecryptionOptions sent in connect/token,
// but as of 2025-12-19 they're both using different casing conventions.
```
— [ciphers.rs#L170-L172](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/ciphers.rs#L170-L172)

### 1.5 Token 响应中的 PascalCase

`/connect/token` 的响应故意使用 PascalCase（Kdf, KdfIterations, KdfMemory, KdfParallelism, Key, PrivateKey, ResetMasterPassword, ForcePasswordReset, MasterPasswordPolicy, UserDecryptionOptions 等），这与 API 层其余部分使用的 camelCase 不同。原因是 Bitwarden 的 Identity 模块（基于 OAuth2 规范）独立演化，客户端在 token 响应解析路径上硬编码了 PascalCase。

```rust
let mut result = json!({
    "access_token": auth_tokens.access_token(),
    "expires_in": auth_tokens.expires_in(),
    "token_type": "Bearer",
    "refresh_token": auth_tokens.refresh_token(),
    "PrivateKey": user.private_key,        // PascalCase
    "Kdf": user.client_kdf_type,           // PascalCase
    "KdfIterations": user.client_kdf_iter,  // PascalCase
    "KdfMemory": user.client_kdf_memory,    // PascalCase
    "KdfParallelism": user.client_kdf_parallelism, // PascalCase
    "ResetMasterPassword": false,
    "ForcePasswordReset": false,
    "MasterPasswordPolicy": master_password_policy, // PascalCase 且 object 为 "masterPasswordPolicy"
    "scope": auth_tokens.scope(),
    "AccountKeys": account_keys,
    "UserDecryptionOptions": {
        "HasMasterPassword": has_master_password,
        "MasterPasswordUnlock": master_password_unlock,
        "Object": "userDecryptionOptions"   // 注意：Object 本身也是 PascalCase
    },
});
```
— [identity.rs#L536-L556](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/identity.rs#L536-L556)

`MasterPasswordPolicy` 字段的内部也使用了 PascalCase 的 `Object` 键名，如 [api/mod.rs#L124](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/mod.rs#L124)：
```rust
mpp_json["Object"] = json!("masterPasswordPolicy"); // 注意: Upstream 仍使用 PascalCase
```

### 1.6 列表响应格式

列表端点（如 `GET /api/ciphers`）使用 `"object": "list"` + `"data"` 数组 + `"continuationToken"` 的格式：

```rust
Ok(Json(json!({
    "data": ciphers_json,
    "object": "list",
    "continuationToken": null
})))
```
— [ciphers.rs#L218-L222](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/ciphers.rs#L218-L222)

### 1.7 Membership 的 Manager → Custom 类型映射 HACK

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
— [organization.rs#L310-L317](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L310-L317)

同时在反序列化时，`"4" | "Custom"` 也会映射回 `MembershipType::Manager`：— [organization.rs#L114-L115](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L114-L115)

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
— [util.rs#L560-L565](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/util.rs#L560-L565)

核心逻辑在 `lcase_first()` 和 `convert_json_key_lcase_first()`：

```rust
pub fn lcase_first(s: &str) -> String {
    let mut c = s.chars();
    match c.next() {
        None => String::new(),
        Some(f) => f.to_lowercase().collect::<String>() + c.as_str(),
    }
}
```
— [util.rs#L373-L379](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/util.rs#L373-L379)

对 `ssn`（社保号）键做了特殊处理，因为 `lcase_first("SSN")` 会变成 `sSN`，需要保持为 `ssn`：
— [util.rs#L621-L628](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/util.rs#L621-L628)

**影响**：Cipher 的 `fields`、`password_history`、`data` 列在 DB 中存储的 JSON，反序列化时都经过 `LowerCase<Value>` 处理，确保无论客户端发来 `Uri` 还是 `uri`、`Match` 还是 `match`，都能正确识别。

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
— [util.rs#L645-L650](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/util.rs#L645-L650)

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
— [cipher.rs#L205-L214](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L205-L214)

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
— [cipher.rs#L267-L275](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L267-L275)

### 2.5 passwordHistory 中 password 为 null 的条目必须过滤

**客户端崩溃风险**：较新版本的 Bitwarden 客户端无法处理 passwordHistory 中 `password` 字段为 `null` 的条目，Vaultwarden 直接过滤掉：

```rust
.filter_map(|d| match d.data.get("password") {
    Some(p) if p.is_string() => Some(d.data),
    _ => None, // null password 的条目被丢弃
})
```
— [cipher.rs#L233-L237](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L233-L237)

### 2.6 type_data 解析失败时必须返回空对象

**移动端崩溃风险**：如果 cipher 的 `data` 字段无法解析为 JSON 对象，不能返回 null，必须回退为空对象 `{}`：

```rust
let mut type_data_json = serde_json::from_str::<LowerCase<Value>>(&self.data)
    .inspect_err(|_| warn!("Error parsing data field for {}", self.uuid))
    .map_or_else(|_| Value::Object(serde_json::Map::new()), |d| d.data);
```
— [cipher.rs#L251-L255](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L251-L255)

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
— [cipher.rs#L285-L294](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L285-L294)

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
— [cipher.rs#L296-L307](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L296-L307)

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
— [cipher.rs#L257-L277](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L257-L277)

### 2.10 data_json 的必含字段

无论 cipher 类型，`data_json`（即 `type_data_json` 的克隆）始终注入四个字段。这同样是因为某些客户端假设这些字段总在 data 内：

```rust
data_json["fields"] = json!(fields_json);
data_json["name"] = json!(self.name);
data_json["notes"] = json!(self.notes);
data_json["passwordHistory"] = Value::Array(password_history_json.clone());
```
— [cipher.rs#L312-L317](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L312-L317)

---

## 3. 错误响应

### 3.1 两套错误格式：ApiErrorResponse 与 CompactApiErrorResponse

Vaultwarden 定义了两套错误响应格式，均通过自定义 `Serialize` 实现以避免在内存中构造包含大量空字段的结构体：

**ApiErrorResponse**（9 个字段）— 用于大多数 API 错误：

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
— [error.rs#L214-L248](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/error.rs#L214-L248)

**CompactApiErrorResponse**（6 个字段）— 用于部分较新的错误响应：

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
— [error.rs#L254-L270](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/error.rs#L254-L270)

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

### 3.2 错误路由机制：Display trait 选择序列化函数

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
— [error.rs#L73-L108](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/error.rs#L73-L108)

**关键点**：`Error::to_string()` 的输出就是 HTTP 响应 body。`Display` trait 实现调用对应的生成函数，这个函数返回序列化后的 JSON 字符串。然后 Rocket 的 `Responder` 实现将其包装为 JSON 响应：

```rust
impl Responder<'_, 'static> for Error {
    fn respond_to(self, _: &Request<'_>) -> response::Result<'static> {
        match self.kind {
            ErrorKind::Empty(_) | ErrorKind::Simple(_) | ErrorKind::Compact(_) => {}
            _ => error!(target: "error", "{self:#?}"),
        }
        let code = Status::from_code(self.code).unwrap_or(Status::BadRequest);
        let body = self.to_string(); // ← Display trait, 即序列化后的 JSON
        Response::build()
            .status(code)
            .header(ContentType::JSON)
            .sized_body(Some(body.len()), Cursor::new(body)).ok()
    }
}
```
— [error.rs#L310-L321](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/error.rs#L310-L321)

### 3.3 三种特殊错误路径

**`err_json!` — 完全自定义 JSON 响应**：用于需要特定格式的场景，如 2FA 错误和 refresh_token 错误。通过 `ErrorKind::Json(Value)` 绕过标准错误格式。

**2FA 错误**（`json_err_twofactor`）的格式与标准 ApiErrorResponse 完全不同，它使用 OAuth2 风格：

```rust
json!({
    "error" : "invalid_grant",
    "error_description" : "Two factor required.",
    "TwoFactorProviders" : [...],
    "TwoFactorProviders2" : { ... },
    "MasterPasswordPolicy": { "Object": "masterPasswordPolicy" }
})
```
— [identity.rs#L906-L914](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/identity.rs#L906-L914)

客户端在收到 `error: "invalid_grant"` 时会检查 `TwoFactorProviders2` 并触发 2FA 流程。如果这里使用标准 ApiErrorResponse 格式，客户端将无法识别 2FA 要求。

**refresh_token 错误**：同样需要 `{"error": "invalid_grant"}` 格式，客户端据此强制登出：
— [identity.rs#L138-L160](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/identity.rs#L138-L160)

**验证错误**（Cipher 批量导入）：使用 `err_json!` 发送带字段级 `validationErrors` 的响应：

```rust
let err_json = json!({
    "message": "The model state is invalid.",
    "validationErrors" : validation_errors, // 如 {"Ciphers[0].Notes": ["..."]}
    "object": "error"
});
err_json!(err_json, "Import validation errors")
```
— [cipher.rs#L131-L137](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/cipher.rs#L131-L137)

### 3.4 404 Catcher

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
— [mod.rs#L262-L271](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/mod.rs#L262-L271)

这是另一种错误格式。客户端对 404 的处理路径与业务错误不同。

---

## 4. 客户端假设与取舍

### 4.1 config 端点的 version 字段

```rust
"version": "2025.12.0",
```
— [mod.rs#L231](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/mod.rs#L231)

源码注释说明了这个版本号的含义：

```rust
// Note: The clients use this version to handle backwards compatibility concerns
// This means they expect a version that closely matches the Bitwarden server version
// We should make sure that we keep this updated when we support the new server features
// Version history:
// - Individual cipher key encryption: 2024.2.0
// - Mobile app support for MasterPasswordUnlockData: 2025.8.0
```

**取舍**：Vaultwarden 必须声明一个与 Bitwarden 服务器版本兼容的版本号，否则客户端会因版本检查而禁用功能或使用错误的代码路径。

### 4.2 featureStates

config 端点中的 `featureStates` 控制客户端功能开关。Vaultwarden 解析管理员配置的 `experimental_client_feature_flags`，同时硬编码启用特定功能：

```rust
let mut feature_states = parse_experimental_client_feature_flags(
    &CONFIG.experimental_client_feature_flags(),
    &FeatureFlagFilter::ValidOnly,
);
feature_states.insert("pm-19148-innovation-archive".to_owned(), true);
```
— [mod.rs#L218-L222](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/mod.rs#L218-L222)

### 4.3 Premium 功能全部解锁

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

### 4.4 组织功能标志的硬编码

Organization::to_json() 和 Membership::to_json() 中大量功能标志硬编码为 false，对应 Vaultwarden 不支持的功能：

| 标志 | 值 | 原因 |
|---|---|---|
| `useScim` | false | 非 AGPLv3 许可 |
| `useSecretsManager` | false | 非 AGPLv3 许可 |
| `useKeyConnector` | false | 不支持 |
| `useSso` | false | Organization::to_json 中；但 Membership::to_json 也为 false |
| `ssoBound` | false | 不支持 |
| `useRiskInsights` | false | 非 AGPLv3 许可 |
| `useAdminSponsoredFamilies` | false | 不支持 |
| `useActivateAutofillPolicy` | false | 不支持 |

动态值的标志：
- `useEvents` → `CONFIG.org_events_enabled()`
- `useGroups` → `CONFIG.org_groups_enabled()`
- `useResetPassword` → `CONFIG.mail_enabled()`

### 4.5 存储容量无限制

```rust
"maxStorageGb": i16::MAX, // The value doesn't matter, we don't check server-side
```
— [organization.rs#L205](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L205)

**取舍**：设置为 `i16::MAX`（32767 GB）而非 null，因为某些客户端在 null 时可能显示为 0 GB 限制。

### 4.6 planType 和 productTierType

```rust
// Organization::to_json()
"planType": 6, // Custom plan

// Membership::to_json()
"productTierType": 3, // Enterprise tier
```

**取舍**：planType=6（Custom）和 productTierType=3（Enterprise）确保客户端不会显示付费相关的 UI 元素（如升级提示）。

### 4.7 seats 硬编码

```rust
// Membership::to_json()
"seats": 20, // hardcoded maxEmailsCount in the web-vault
```
— [organization.rs#L474](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L474)

Web Vault 使用 `seats` 值作为邀请用户时的最大邮件数上限。

### 4.8 SSH Key 按客户端版本过滤

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
— [ciphers.rs#L128-L137](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/ciphers.rs#L128-L137)

**取舍**：如果客户端版本低于 2024.12.0，完全移除 SSH Key 类型的 cipher。无法获取版本信息时也默认不显示——宁可丢失数据也不让旧客户端崩溃。

### 4.9 2FA Email 发送行为的版本控制

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
— [identity.rs#L972-L983](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/identity.rs#L972-L983)

**取舍**：2025.5.0+ 的客户端会自己调用专门的 email 发送端点，服务端不应重复发送。

### 4.10 webauthn 端点空响应

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
— [mod.rs#L199-L208](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/core/mod.rs#L199-L208)

**取舍**：Vaultwarden 暂不支持 passkey 登录，但必须提供此端点。返回 404 会导致客户端密钥轮换问题，返回空列表则安全通过。

### 4.11 MasterPasswordPolicy 的 PascalCase Object 键

```rust
// NOTE: Upstream still uses PascalCase here for `Object`!
mpp_json["Object"] = json!("masterPasswordPolicy");
```
— [api/mod.rs#L124](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/mod.rs#L124)

这与 API 层其余部分使用 camelCase 的 `object` 键不同，是上游的遗留不一致。

### 4.12 MasterPasswordUnlockData 的字段名过渡

```rust
// This field is named inconsistently and will be removed and replaced by the "wrapped" variant in the apps.
// https://github.com/bitwarden/android/blob/release/2025.12-rc41/network/src/main/kotlin/com/bitwarden/network/model/MasterPasswordUnlockDataJson.kt#L22-L26
"MasterKeyEncryptedUserKey": user.akey,
"MasterKeyWrappedUserKey": user.akey,
```
— [identity.rs#L664-L668](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/api/identity.rs#L664-L668)

**取舍**：同时输出新旧两个键名（值相同），确保新旧客户端都能正确解密。旧键名 `MasterKeyEncryptedUserKey` 计划在未来被移除。

### 4.13 limitCollectionCreation 按 MembershipType 控制

```rust
// Membership::to_json()
"limitCollectionCreation": self.atype < MembershipType::Manager || !self.access_all,
```
— [organization.rs#L513](file:///d:/fz/0601/solo-dogfeeding/code/19-vaultwarden/src/db/models/organization.rs#L513)

**取舍**：将集合创建权限限制在 Manager 及以上级别且具有 access_all 权限的用户，防止权限问题。而 Organization::to_json() 中 `limitCollectionCreation` 始终为 true，因为组织级别的设置不同。

---

## 5. 总结

Vaultwarden 的 API 兼容层设计可归纳为以下原则：

1. **响应对象通过 `object` 鉴别器组织**：每个 JSON 响应都带 `object` 字段标识类型，客户端据此分派反序列化逻辑。同一模型支持多级响应（collection/collectionDetails），Vaultwarden 的取舍是始终返回最详细的级别。

2. **字段容错的核心动机是防止客户端崩溃**：fields.type 为字符串→崩溃；type_data 为 null→崩溃；SecureNote 缺少 type→崩溃；SSH Key 缺必填字段→崩溃。这些修正不是"锦上添花"而是"不做就崩"，说明 Bitwarden 客户端在反序列化路径上缺乏防御性编程。

3. **错误响应有三种互不兼容的格式**：标准 ApiErrorResponse（9字段）、CompactApiErrorResponse（6字段）、以及 OAuth2 风格的 2FA/refresh_token 错误。客户端在三条不同代码路径上分别处理这三类错误。

4. **客户端假设的取舍倾向于"安全优先"**：无法确定客户端版本时默认不显示 SSH Key；不支持的功能端点返回空数据而非 404；Premium 功能全部解锁以避免功能被错误禁用；版本号必须与上游同步以通过客户端的兼容性检查。
