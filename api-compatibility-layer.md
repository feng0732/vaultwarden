# Vaultwarden Bitwarden API 兼容层分析

本文档从代码实现角度梳理 Vaultwarden 与 Bitwarden API 兼容层的设计与实现。

## 目录
- [1. 模型转换 (Model Conversion)
- [2. 字段兼容 (Field Compatibility)
- [3. 错误响应 (Error Responses)
- [4. 客户端假设 (Client Assumptions)

---

## 1. 模型转换 (Model Conversion)

### 1.1 核心转换模式

Vaultwarden 采用数据库模型通过 `to_json()` 方法族将内部数据模型转换为 Bitwarden 客户端期望的 API 响应格式。

#### 转换位置: `src/db/models/` 目录下各模型的 `to_json()` 方法

### 1.2 Cipher 模型转换 (cipher.rs#L145-L412)

**Ciper::to_json()** 是最复杂的转换逻辑，包含：

- **附件处理**：将附件列表转换为 JSON 数组
- **访问权限计算**：read_only, hide_passwords, manage 权限
- **字段类型修正**：fields 字段类型转换（字符串转数字）
- **密码历史过滤**：过滤无效的 null 值密码历史条目
- **类型数据处理**：根据 cipher 类型（Login/SecureNote/Card/Identity/SshKey）处理
- **集合 ID**：collectionIds 字段填充
- **用户特定字段**：folderId, favorite, archivedDate, edit, viewPassword, permissions

```rust
// 核心转换结构
json!({
    "object": "cipherDetails",  // 响应类型标识
    "id": self.uuid,
    "type": self.atype,
    "creationDate": format_date(&self.created_at),
    "revisionDate": format_date(&self.updated_at),
    "deletedDate": self.deleted_at.map_or(Value::Null, |d| Value::String(format_date(&d))),
    "reprompt": self.reprompt,
    "organizationId": self.organization_uuid,
    "key": self.key,
    "attachments": attachments_json,
    "collectionIds": collection_ids,
    "name": self.name,
    "notes": self.notes,
    "fields": fields_json,
    "data": data_json,
    "passwordHistory": password_history_json,
    // 类型特定字段
    "login": null, "secureNote": null, "card": null, "identity": null, "sshKey": null,
    // 用户同步特定字段
    "folderId": folder_id,
    "favorite": favorite,
    "archivedDate": archived_date,
    "edit": !read_only,
    "viewPassword": !hide_passwords,
    "permissions": {
        "delete": !read_only,
        "restore": !read_only,
    },
})
```

### 1.3 User 模型转换 ([user.rs#L256-L293)

**User::to_json()** 包含：

- 组织成员关系转换
- 双因素认证状态
- 用户状态（Enabled/Invited）
- Premium 状态（始终为 true）
- 文化设置（默认为 en-US）

```rust
json!({
    "_status": status as i32,
    "id": self.uuid,
    "name": self.name,
    "email": self.email,
    "emailVerified": !CONFIG.mail_enabled() || self.verified_at.is_some(),
    "premium": true,  // 始终为 true
    "premiumFromOrganization": false,
    "culture": "en-US",
    "twoFactorEnabled": twofactor_enabled,
    "key": self.akey,
    "privateKey": self.private_key,
    "securityStamp": self.security_stamp,
    "organizations": orgs_json,
    "providers": [],
    "providerOrganizations": [],
    "forcePasswordReset": false,
    "avatarColor": self.avatar_color,
    "usesKeyConnector": false,
    "creationDate": format_date(&self.created_at),
    "object": "profile",
})
```

### 1.4 Organization 模型转换 ([organization.rs#L199-L246)

**Organization::to_json()** 包含大量功能标志字段：

```rust
json!({
    "id": self.uuid,
    "name": self.name,
    "seats": null,
    "maxCollections": null,
    "maxStorageGb": i16::MAX,  // 无实际限制
    "use2fa": true,
    "useCustomPermissions": true,
    "useDirectory": false,
    "useEvents": CONFIG.org_events_enabled(),
    "useGroups": CONFIG.org_groups_enabled(),
    "useTotp": true,
    "usePolicies": true,
    "useScim": false,  // 不支持
    "useSso": false,   // 不支持
    "useKeyConnector": false,
    "usePasswordManager": true,
    "useSecretsManager": false,  // 不支持
    "selfHost": true,
    "useApi": true,
    "hasPublicAndPrivateKeys": self.private_key.is_some() && self.public_key.is_some(),
    "useResetPassword": CONFIG.mail_enabled(),
    "allowAdminAccessToAllCollectionItems": true,
    "limitCollectionCreation": true,
    "limitCollectionDeletion": true,
    "billingEmail": self.billing_email,
    "planType": 6,  // Custom plan
    "usersGetPremium": true,
    "object": "organization",
})
```

### 1.5 Collection 模型转换 ([collection.rs#L68-L147)

**Collection::to_json()** 和 **to_json_details()**：

```rust
// 基础转换
json!({
    "externalId": self.external_id,
    "id": self.uuid,
    "organizationId": self.org_uuid,
    "name": self.name,
    "object": "collection",
})

// 详情转换（含权限）
json_object["object"] = json!("collectionDetails");
json_object["readOnly"] = json!(read_only);
json_object["hidePasswords"] = json!(hide_passwords);
json_object["manage"] = json!(manage);
```

### 1.6 同步响应结构 ([ciphers.rs#L121-L204)

**sync() 端点返回完整同步数据：

```rust
json!({
    "profile": user_json,
    "folders": folders_json,
    "collections": collections_json,
    "policies": policies_json,
    "ciphers": ciphers_json,
    "domains": domains_json,
    "sends": sends_json,
    "userDecryption": {
        "masterPasswordUnlock": master_password_unlock,
    },
    "object": "sync"
})
```

---

## 2. 字段兼容 (Field Compatibility)

### 2.1 大小写不敏感反序列化 ([util.rs#L560-L620)

**LowerCase<T> 包装器实现字段名大小写不敏感反序列化：

```rust
#[derive(Serialize, Deserialize)]
pub struct LowerCase<T: DeserializeOwned> {
    #[serde(deserialize_with = "lowercase_deserialize")]
    #[serde(flatten)]
    pub data: T,
}
```

**用途**：处理 Bitwarden 客户端发送的 PascalCase、camelCase、snake_case 等不一致字段名。

### 2.2 字段别名 (Field Aliases)

使用 `#[serde(alias = "...")] 处理字段名变体：

```rust
// KDFData ([accounts.rs#L81-L92)
#[derive(Debug, Deserialize, Eq, PartialEq)]
#[serde(rename_all = "camelCase")]
pub struct KDFData {
    #[serde(alias = "kdfType")]
    kdf: i32,
    #[serde(alias = "iterations")]
    kdf_iterations: i32,
    #[serde(alias = "memory")]
    kdf_memory: Option<i32>,
    #[serde(alias = "parallelism")]
    kdf_parallelism: Option<i32>,
}

// RegisterData ([accounts.rs#L94-L120)
#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct RegisterData {
    #[serde(flatten)]
    kdf: KDFData,
    #[serde(alias = "userSymmetricKey")]
    key: String,
    #[serde(alias = "userAsymmetricKeys")]
    keys: Option<KeysData>,
    master_password_hash: String,
    #[serde(alias = "token")]
    org_invite_token: Option<String>,
}
```

### 2.3 日期格式标准化

**format_date()** 函数确保日期格式符合 Bitwarden 期望：

```rust
// util.rs
pub fn format_date(date: &NaiveDateTime) -> String {
    // 转换为 ISO 8601 格式
}

pub fn validate_and_format_date(date_str: &str) -> String {
    // 验证并标准化日期格式
}
```

### 2.4 字段类型修正

**Cipher::to_json()** 中的类型修正逻辑：

```rust
// 字段类型转换 - fields 字段 type 必须是数字
match f.data.get("type") {
    Some(t) if t.is_number() => {}
    Some(t) if t.is_string() => {
        let type_num = &t.as_str().unwrap_or("1").parse::<u8>().unwrap_or(1);
        f.data["type"] = json!(type_num);
    }
    _ => {
        f.data["type"] = json!(1);
    }
}

// URI match 字段类型修正
if uri["match"].is_string() => {
    let match_value = match uri["match"].as_str().unwrap_or_default().parse::<u8>() {
        Ok(n) => json!(n),
        _ => Value::Null,
    };
    uri["match"] = match_value;
}

// 密码历史日期格式修正
let lud = if let Some(l) = d.get("lastUsedDate").and_then(|l| l.as_str()) {
    validate_and_format_date(l)
} else {
    "1970-01-01T00:00:00.000000Z".to_owned()
};
d["lastUsedDate"] = json!(lud);
```

### 2.5 空值与默认值处理

```rust
// type_data 默认值 - 空对象防止移动端崩溃
let mut type_data_json = serde_json::from_str::<LowerCase<Value>>(&self.data)
    .map_or_else(|_| Value::Object(serde_json::Map::new()), |d| d.data);

// SecureNote 类型修复
if self.atype == 2 {
    match type_data_json {
        Value::Object(ref t) if t.get("type").is_some_and(Value::is_number) => {}
        _ => {
            type_data_json = json!({"type": 0});
        }
    }
}

// SSH Key 无效数据处理
if self.atype == 5
    && (type_data_json["keyFingerprint"].as_str().is_none_or(str::is_empty)
        || type_data_json["privateKey"].as_str().is_none_or(str::is_empty)
        || type_data_json["publicKey"].as_str().is_none_or(str::is_empty))
{
    type_data_json = Value::Null;
}
```

### 2.6 向后兼容字段

```rust
// Login 类型 URI 向后兼容
if self.atype == 1 {
    type_data_json["uri"] = Value::Null;
    if let Some(uris) = type_data_json["uris"].as_array_mut()
        && !uris.is_empty()
    {
        // 修复 uri match 值
        for uri in &mut *uris {
            if uri["match"].is_string() {
                // 字符串转数字
            }
        }
        type_data_json["uri"] = uris[0]["uri"].clone();
    }
}
```

---

## 3. 错误响应 (Error Responses)

### 3.1 错误响应结构 ([error.rs#L214-L297)

**ApiErrorResponse** 自定义序列化实现 Bitwarden 兼容错误格式：

```rust
// 完整错误响应结构（9个字段）
impl Serialize for ApiErrorResponse<'_> {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut state = serializer.serialize_struct("ApiErrorResponse", 9)?;

        state.serialize_field("message", self.0.message)?;

        let mut validation_errors = std::collections::HashMap::with_capacity(1);
        validation_errors.insert("", vec![self.0.message]);
        state.serialize_field("validationErrors", &validation_errors)?;

        let error_model = ErrorModel {
            message: self.0.message,
            object: "error",
        };
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

**CompactApiErrorResponse** 简化版（6个字段）：

```rust
impl Serialize for CompactApiErrorResponse<'_> {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
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

### 3.2 错误响应示例

**标准错误响应 JSON 格式：

```json
{
  "message": "错误消息",
  "validationErrors": {
    "": ["错误消息"]
  },
  "errorModel": {
    "message": "错误消息",
    "object": "error"
  },
  "error": "",
  "error_description": "",
  "exceptionMessage": null,
  "exceptionStackTrace": null,
  "innerExceptionMessage": null,
  "object": "error"
}
```

### 3.3 错误类型宏定义 ([error.rs#L73-L108)

```rust
make_error! {
    Empty(Empty):     no_source, serialize,
    Simple(String):  no_source,  api_error,
    Compact(Compact):  no_source,  compact_api_error,
    CustomHttpClient(CustomHttpClientError): has_source, api_error,
    Json(Value):           no_source,  serialize,
    Db(DieselErr):         has_source, api_error,
    R2d2(R2d2Err):         has_source, api_error,
    Serde(SerdeErr):       has_source, api_error,
    JWt(JwtErr):           has_source, api_error,
    // ... 更多错误类型
}
```

### 3.4 错误宏 ([error.rs#L326-L422)

```rust
// 标准错误宏
err!("错误消息")
err!("用户消息", "日志消息")
err_code!("错误消息", 404)

// 静默错误（不记录日志）
err_silent!("错误消息")

// JSON 错误（自定义错误响应）
err_json!(json_value, "日志消息")

// 验证错误示例（cipher.rs#L130-L137)
let err_json = json!({
    "message": "The model state is invalid.",
    "validationErrors" : validation_errors,
    "object": "error"
});
err_json!(err_json, "Import validation errors")
```

### 3.5 Rocket Responder 实现 ([error.rs#L310-L321)

```rust
impl Responder<'_, 'static> for Error {
    fn respond_to(self, _: &Request<'_>) -> response::Result<'static> {
        // 记录错误日志（除 Empty/Simple/Compact 外）
        match self.kind {
            ErrorKind::Empty(_) | ErrorKind::Simple(_) | ErrorKind::Compact(_) => {}
            _ => error!(target: "error", "{self:#?}"),
        }

        let code = Status::from_code(self.code).unwrap_or(Status::BadRequest);
        let body = self.to_string();
        Response::build()
            .status(code)
            .header(ContentType::JSON)
            .sized_body(Some(body.len()), Cursor::new(body)).ok()
    }
}
```

---

## 4. 客户端假设 (Client Assumptions)

### 4.1 客户端版本检测

**基于版本过滤 SSH Key 显示逻辑 ([ciphers.rs#L128-L137)

```rust
// 过滤 SSH Key 仅在客户端版本 >= 2024.12.0 及以上显示
let show_ssh_keys = if let Some(client_version) = client_version {
    let ver_match = semver::VersionReq::parse(">=2024.12.0").unwrap();
    ver_match.matches(&client_version.0)
} else {
    false
};
if !show_ssh_keys {
    ciphers.retain(|c| c.atype != 5);
}
```

### 4.2 设备类型与刷新令牌有效期 ([auth.rs#L39-L41)

```rust
pub static DEFAULT_REFRESH_VALIDITY: LazyLock<TimeDelta> = LazyLock::new(|| TimeDelta::try_days(30).unwrap());
pub static MOBILE_REFRESH_VALIDITY: LazyLock<TimeDelta> = LazyLock::new(|| TimeDelta::try_days(90).unwrap());
```

**设备类型映射：

- **桌面端/浏览器/CLI：30 天
- **移动端**：90 天

### 4.3 JWT Token 过期时间 ([auth.rs#L37)

```rust
// Bitwarden 认为 token 过期的时间限制
pub static BW_EXPIRATION: LazyLock<TimeDelta> = LazyLock::new(|| TimeDelta::try_minutes(5).unwrap());
```

### 4.4 Cipher 响应类型假设

**Bitwarden 有三种 cipher 响应模型：

1. **cipherMini** - 最小信息
2. **cipher** - 标准信息
3. **cipherDetails** - 详细信息（Vaultwarden 目前只支持 cipherDetails 类型，客户端会忽略多余字段。

### 4.5 客户端字段存在性假设

```rust
// data_json 必须包含的字段（所有类型都需要
data_json["fields"] = json!(fields_json);
data_json["name"] = json!(self.name);
data_json["notes"] = json!(self.notes);
data_json["passwordHistory"] = Value::Array(password_history_json.clone());
```

### 4.6 空数组/空对象假设

```rust
// 空对象防止移动端崩溃
let mut type_data_json = serde_json::from_str::<LowerCase<Value>>(&self.data)
    .map_or_else(|_| Value::Object(serde_json::Map::new()), |d| d.data);
```

### 4.7 组织功能标志默认值

```rust
// Organization::to_json() 中的默认值：
"use2fa": true,
"useTotp": true,
"usePolicies": true,
"usersGetPremium": true,
"maxStorageGb": i16::MAX,  // 无限制
"planType": 6,  // Custom plan
```

### 4.8 用户 Premium 状态

```rust
// 用户始终被视为 Premium 用户
"premium": true,
```

### 4.9 WebSocket 通知假设

```rust
// notifications/hub 端点的特殊处理
if req_uri_path.ends_with("notifications/hub") || req_uri_path.ends_with("notifications/anonymous-hub") {
    // WebSocket 连接不添加安全头
}
```

### 4.10 CSP 框架祖先

```rust
// CSP frame-ancestors 允许官方扩展 ID：
chrome-extension://nngceckbapebfimnlniiiahkandclblb  // Chrome
chrome-extension://jbkfoedolllekgbhcbcoahefnbanhhlh?hl=en-US  // Edge
moz-extension://*  // Firefox
```

---

## 总结

Vaultwarden 的 API 兼容层通过以下方式实现与 Bitwarden 客户端的兼容性：

1. **模型转换层**：通过 `to_json()` 方法将内部模型转换为 Bitwarden API 格式
2. **字段兼容**：大小写不敏感、类型修正、默认值填充
3. **错误响应**：严格遵循 Bitwarden 错误格式（9字段标准响应结构
4. **客户端假设**：版本检测、设备类型检测、功能标志默认值

这些设计确保了 Vaultwarden 可以无缝对接官方 Bitwarden 客户端（网页版、桌面版、移动端、浏览器扩展等）。
