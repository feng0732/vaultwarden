# Vaultwarden Bitwarden 导入导出格式兼容性分析

## 目录

1. [概述](#概述)
2. [格式识别与导入端点](#格式识别与导入端点)
3. [字段映射关系](#字段映射关系)
4. [错误处理机制](#错误处理机制)
5. [数据写入边界与约束](#数据写入边界与约束)
6. [导出格式](#导出格式)
7. [代码参考](#代码参考)

---

## 概述

Vaultwarden 作为 Bitwarden 的兼容服务端实现，提供了完整的个人保险箱和组织保险箱导入导出功能。导入导出基于 Bitwarden 官方 JSON 格式，涉及 Cipher（密码条目）、Folder（文件夹）、Collection（集合）及其关联关系。

---

## 格式识别与导入端点

### 两个核心导入端点

Vaultwarden 支持两种导入场景，分别对应两个独立的 API 端点：

| 端点 | 用途 | 代码位置 |
|------|------|----------|
| `POST /ciphers/import` | 个人保险箱导入 | [ciphers.rs#L595](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L595-L644) |
| `POST /ciphers/import-organization` | 组织保险箱导入 | [organizations.rs#L1782](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L1782-L1866) |

### 导入数据结构

#### 个人导入结构 (`ImportData`)

```rust
struct ImportData {
    ciphers: Vec<CipherData>,       // 密码条目列表
    folders: Vec<FolderData>,       // 文件夹列表
    folder_relationships: Vec<RelationsData>,  // 条目-文件夹关系
}
```
位置：[ciphers.rs#L578-L594](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L578-L594)

#### 组织导入结构 (`ImportData`)

```rust
struct ImportData {
    ciphers: Vec<CipherData>,                   // 密码条目列表
    collections: Vec<FullCollectionData>,       // 集合列表
    collection_relationships: Vec<RelationsData>, // 条目-集合关系
}
```
位置：[organizations.rs#L1764-L1780](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L1764-L1780)

#### 关联关系结构 (`RelationsData`)

```rust
struct RelationsData {
    key: usize,    // Cipher 在数组中的索引
    value: usize,  // Folder/Collection 在数组中的索引
}
```

**关键设计**：使用 **数组索引** 而非 UUID 来建立条目与文件夹/集合的关联，这是 Bitwarden 官方格式的标准做法。

- 个人导入：一个 Cipher 只能属于 **一个** Folder（通过 HashMap 覆盖实现）
- 组织导入：一个 Cipher 可以属于 **多个** Collection（通过 Vec 存储）

### Cipher 类型识别

导入支持 5 种 Cipher 类型，通过 `type` 字段的整数值识别：

| 类型值 | 类型名称 | 对应数据字段 |
|--------|----------|-------------|
| 1 | Login（登录） | `login` |
| 2 | SecureNote（安全笔记） | `secure_note` |
| 3 | Card（银行卡） | `card` |
| 4 | Identity（身份） | `identity` |
| 5 | SshKey（SSH 密钥） | `ssh_key` |

映射代码位置：[ciphers.rs#L505-L524](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L505-L524)

---

## 字段映射关系

### CipherData → Cipher 核心字段映射

`CipherData` 是客户端提交的导入数据结构，`Cipher` 是数据库存储模型。映射逻辑集中在 `update_cipher_from_data()` 函数中：

| CipherData 字段 | Cipher 字段 | 说明 |
|----------------|-------------|------|
| `type` | `atype` | 条目类型（1-5） |
| `name` | `name` | 条目名称（必填） |
| `notes` | `notes` | 备注（可选，受长度限制） |
| `key` | `key` | 加密密钥（可选） |
| `fields` | `fields` | 自定义字段（JSON 序列化存储） |
| `login/secure_note/card/identity/ssh_key` | `data` | 类型特定数据（JSON 序列化存储） |
| `password_history` | `password_history` | 密码历史（JSON 序列化存储） |
| `reprompt` | `reprompt` | 重新提示类型（0=无，1=密码） |
| `folder_id` | - | 不直接存储，通过 FolderCipher 关联表 |
| `organization_id` | `organization_uuid` | 组织 ID（个人条目设 user_uuid） |
| `favorite` | - | 通过 Favorite 关联表存储 |
| `archived_date` | - | 通过 Archive 关联表存储 |

映射核心代码：[ciphers.rs#L395-L576](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L395-L576)

### CipherData 结构详情

```rust
pub struct CipherData {
    pub id: Option<CipherId>,
    #[serde(deserialize_with = "deser_opt_nonempty_str")]
    pub folder_id: Option<FolderId>,
    #[serde(alias = "organizationID")]
    pub organization_id: Option<OrganizationId>,

    key: Option<String>,
    pub r#type: i32,
    pub name: String,
    pub notes: Option<String>,
    fields: Option<Value>,

    login: Option<Value>,
    secure_note: Option<Value>,
    card: Option<Value>,
    identity: Option<Value>,
    ssh_key: Option<Value>,

    favorite: Option<bool>,
    reprompt: Option<i32>,

    pub password_history: Option<Value>,
    attachments: Option<Value>,
    attachments2: Option<HashMap<AttachmentId, Attachments2Data>>,

    last_known_revision_date: Option<String>,
    archived_date: Option<String>,
}
```
位置：[ciphers.rs#L249-L301](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L249-L301)

**字段反序列化兼容处理**：

1. **`folder_id` 的空字符串兼容**：使用 `deser_opt_nonempty_str` 反序列化器，将空字符串 `""` 转换为 `None`，避免无效的空 ID。
   位置：[util.rs#L630-L643](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/util.rs#L630-L643)

2. **`organization_id` 的大小写兼容**：通过 `#[serde(alias = "organizationID")]` 同时支持 `organizationId` 和 `organizationID` 两种 JSON 键名。

### 数据清洗（clean_cipher_data）

在字段映射前，会对 `fields` 和 `type_data` 进行特殊清洗：

```rust
fn clean_cipher_data(mut json_data: Value) -> Value {
    if json_data.is_array() {
        json_data.as_array_mut().unwrap().iter_mut().for_each(|ref mut f| {
            f.as_object_mut().unwrap().remove("response");
        });
    }
    json_data
}
```

**目的**：移除 JavaScript 客户端产生的冗余 `"response"` 键。该键由客户端 JS 生成，不属于 Bitwarden 标准数据格式，需要在保存前剔除。

清洗代码位置：[ciphers.rs#L409-L416](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L409-L416)

### 文件夹与集合映射

#### 文件夹（Folder）

导入数据结构：
```rust
pub struct FolderData {
    pub name: String,
    #[serde(deserialize_with = "deser_opt_nonempty_str")]
    pub id: Option<FolderId>,
}
```
位置：[folders.rs#L39-L45](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/folders.rs#L39-L45)

**匹配逻辑**：
- 若 `id` 存在于用户现有文件夹集合中 → 复用现有文件夹
- 否则 → 创建新文件夹

#### 集合（Collection）

导入数据结构：
```rust
struct FullCollectionData {
    name: String,
    groups: Vec<CollectionGroupData>,
    users: Vec<CollectionMembershipData>,
    id: Option<CollectionId>,
    external_id: Option<String>,
}
```
位置：[organizations.rs#L128-L136](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L128-L136)

**匹配逻辑**：
- 若 `id` 存在于组织现有集合中 → 复用现有集合（需权限验证）
- 否则 → 创建新集合（需创建集合权限）

### 所有权归属判断

```rust
if let Some(org_id) = data.organization_id {
    // 组织条目
    cipher.organization_uuid = Some(org_id);
    cipher.user_uuid = None;
} else {
    // 个人条目
    cipher.user_uuid = Some(headers.user.uuid.clone());
}
```
位置：[ciphers.rs#L449-L471](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L449-L471)

---

## 错误处理机制

### 前置验证（Pre-validation）

导入采用 **"全有或全无"** 策略：在写入任何数据之前，先对所有 Cipher 进行验证，任何一项不合法则整体失败。

```rust
Cipher::validate_cipher_data(&data.ciphers)?;
```
位置：[ciphers.rs#L605](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L605)
和 [organizations.rs#L1800](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L1800)

### 验证规则

`Cipher::validate_cipher_data()` 检查以下内容：

#### 1. Notes 字段长度限制

```rust
if let Some(note) = &cipher.notes
    && note.len() > max_note_size
{
    validation_errors.insert(
        format!("Ciphers[{index}].Notes"),
        json!(["The field Notes exceeds the maximum encrypted value length of ..."])
    );
}
```

- 默认限制：**10,000 字符**
- 配置 `increase_note_size_limit=true` 时：**100,000 字符**（警告：可能导致客户端问题，且导出不兼容 Bitwarden 官方服务端）

位置：[cipher.rs#L97-L140](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/cipher.rs#L97-L140)
和 [config.rs#L782-L786](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/config.rs#L782-L786)

#### 2. 密码历史 null 值检查

```rust
if let Some(Value::Array(password_history)) = &cipher.password_history {
    for pwh in password_history {
        if let Value::Object(pwo) = pwh
            && pwo.get("password").is_some_and(|p| !p.is_string())
        {
            validation_errors.insert(
                format!("Ciphers[{index}].Notes"),
                json!(["The password history contains a `null` value. Only strings are allowed."])
            );
            break;
        }
    }
}
```

**注意**：这里有一个已知的键名 bug —— 错误键名写为 `"Ciphers[{index}].Notes"` 而非 `"Ciphers[{index}].PasswordHistory"`。

位置：[cipher.rs#L111-L127](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/cipher.rs#L111-L127)

### 验证失败响应格式

验证失败时返回结构化错误 JSON（Bitwarden 标准格式）：

```json
{
    "message": "The model state is invalid.",
    "validationErrors": {
        "Ciphers[0].Notes": ["The field Notes exceeds the maximum encrypted value length of 10000 characters."],
        "Ciphers[1].Notes": ["The password history contains a `null` value. Only strings are allowed."]
    },
    "object": "error"
}
```

### 过程中错误处理

#### 1. 权限错误

| 场景 | 错误消息 |
|------|---------|
| 个人所有权策略生效 | `"Due to an Enterprise Policy, you are restricted from saving items to your personal vault."` |
| 无组织添加权限 | `"You don't have permission to add item to organization"` |
| 无集合创建权限 | `"The current user isn't allowed to create new collections"` |
| 无集合管理权限 | `"The current user isn't allowed to manage this collection"` |
| 文件夹不存在 | `"Invalid folder", "Folder does not exist or belongs to another user"` |

#### 2. 数据错误

| 场景 | 错误消息 |
|------|---------|
| Cipher 类型无效（非 1-5） | `"Invalid type"` |
| 类型数据缺失 | `"Data missing"` |
| 组织 ID 不匹配 | `"Organization mismatch. Please resync the client before updating the cipher"` |
| 客户端数据过期 | `"The client copy of this cipher is out of date. Resync the client and try again."` |

#### 3. 导入时的特殊处理

在导入流程中（`UpdateType::None`），**跳过** 修订日期检查（`last_known_revision_date`），避免因日期不一致导致导入失败：

```rust
if ut != UpdateType::None
    && let Some(dt) = data.last_known_revision_date
{
    // 仅在非导入时检查
}
```
位置：[ciphers.rs#L420-L433](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L420-L433)

---

## 数据写入边界与约束

### 事务边界

**重要**：Vaultwarden 导入流程 **不在单个数据库事务** 中执行。

1. 前置验证通过后开始写入
2. 文件夹/集合逐个创建或复用
3. Cipher 逐个写入
4. 关联关系逐个建立

**风险**：如果中途某个 Cipher 写入失败，已写入的部分不会回滚。但前置验证会拦截绝大多数数据错误。

### 组织导入中的容错

在组织导入流程中，单个 Cipher 写入失败不会中断整体导入：

```rust
update_cipher_from_data(
    &mut cipher, cipher_data, &headers,
    Some(collections.clone()), &conn, &nt, UpdateType::None
).await.ok();  // 使用 .ok() 忽略错误
```
位置：[organizations.rs#L1843-L1853](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L1843-L1853)

相比之下，个人导入遇到错误会立即中止：

```rust
update_cipher_from_data(
    &mut cipher, cipher_data, &headers, None, &conn, &nt, UpdateType::None
).await?;  // 使用 ? 传播错误
```
位置：[ciphers.rs#L636](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L636)

### 权限边界

#### 个人导入权限检查
- 检查个人所有权策略（PersonalOwnership Policy）
- 验证 Folder 归属当前用户

#### 组织导入权限检查
- 组织 ID 匹配：`org_id != headers.membership.org_uuid`
- 已有集合：需确认用户可管理该集合（非 Owner/Admin 时额外检查）
- 新建集合：Manager 且无 `full_access` 权限时禁止创建

### 数据约束

| 约束项 | 限制值 | 说明 |
|--------|--------|------|
| Notes 最大长度 | 10,000 / 100,000 | 可配置，默认 10KB |
| Cipher 类型范围 | 1-5 | Login=1, SecureNote=2, Card=3, Identity=4, SshKey=5 |
| Reprompt 有效值 | 0, 1 | 0=None, 1=Password，其他值被过滤为 None |
| 单 Cipher 文件夹数 | 1 | Cipher 只能属于一个 Folder |
| 单 Cipher 集合数 | N | 可属于多个 Collection |
| 组织导入 folder_id | 忽略 | 强制清空为 None |

### Reprompt 字段过滤

```rust
cipher.reprompt = data.reprompt.filter(
    |r| *r == RepromptType::None as i32 || *r == RepromptType::Password as i32
);
```
只有值为 0 或 1 时才会保存，其他值被静默丢弃。

位置：[ciphers.rs#L532](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L532)

### 修订日期更新

导入完成后，会更新用户/组织的修订版本号以触发客户端同步：

- 个人导入：`user.update_revision(&conn)` + `nt.send_user_update(UpdateType::SyncVault, ...)`
- 组织导入：`user.update_revision(&conn)`

---

## 导出格式

### 两个导出端点

| 端点 | 用途 | 代码位置 |
|------|------|----------|
| `GET /sync` | 个人完整数据同步（含导出所需全部数据） | [ciphers.rs#L121](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L121-L204) |
| `GET /organizations/<org_id>/export` | 组织保险箱导出 | [organizations.rs#L3101](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L3101-L3111) |

### 组织导出格式

```json
{
    "collections": [
        {
            "externalId": null,
            "id": "uuid",
            "organizationId": "uuid",
            "name": "集合名称",
            "object": "collection"
        }
    ],
    "ciphers": [
        // CipherDetails 格式
    ]
}
```

**关键兼容处理**：导出时所有 JSON key 的首字母会被转换为小写（`convert_json_key_lcase_first`），因为客户端无法处理大写首字母的 key。

位置：[organizations.rs#L3095-L3110](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L3095-L3110)

### Key 大小写转换规则

`convert_json_key_lcase_first()` 递归遍历整个 JSON 结构：

```rust
fn process_json_key(key: &str) -> String {
    match key.to_lowercase().as_ref() {
        "ssn" => "ssn".into(),   // 特殊处理：SSN 保持全小写
        _ => lcase_first(key),   // 其余首字母小写
    }
}
```

位置：[util.rs#L621-L628](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/util.rs#L621-L628)

### Cipher 导出格式（CipherDetails）

`Cipher::to_json()` 生成完整的 CipherDetails 响应，包含向后兼容处理：

| 兼容项 | 处理方式 |
|--------|---------|
| Login Uri match 类型 | 字符串转为数字，失败则为 null |
| Login uri 字段 | 取 uris[0].uri 作为兼容字段 |
| passwordRevisionDate | 日期格式校验和修正 |
| SecureNote type | 缺失或非数字时，默认设为 `{"type": 0}` |
| SSH Key 字段完整性 | 缺关键字段时 type_data 设为 null |
| Fields type 字段 | 字符串转数字，默认值 1（隐藏类型） |
| Password History | 过滤 null 密码值，修正 lastUsedDate 格式 |

位置：[cipher.rs#L145-L412](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/cipher.rs#L145-L412)

### Cipher 导出响应字段

```json
{
    "object": "cipherDetails",
    "id": "uuid",
    "type": 1,
    "creationDate": "ISO8601",
    "revisionDate": "ISO8601",
    "deletedDate": null,
    "reprompt": 0,
    "organizationId": null,
    "key": null,
    "attachments": null,
    "organizationUseTotp": true,
    "collectionIds": [],
    "name": "名称",
    "notes": "备注",
    "fields": [],
    "data": {
        "fields": [],
        "name": "名称",
        "notes": "备注",
        "passwordHistory": [],
        // 类型特定字段
        "uris": [],
        "username": "...",
        "password": "..."
    },
    "passwordHistory": [],
    "login": { /* 类型数据 */ },
    "secureNote": null,
    "card": null,
    "identity": null,
    "sshKey": null,
    "folderId": null,
    "favorite": false,
    "archivedDate": null,
    "edit": true,
    "viewPassword": true,
    "permissions": {
        "delete": true,
        "restore": true
    }
}
```

### 用户同步格式（/sync）

个人同步返回更完整的数据结构，客户端可直接用于本地导出：

```json
{
    "profile": { /* 用户信息 */ },
    "folders": [],
    "collections": [],
    "policies": [],
    "ciphers": [],
    "domains": {},
    "sends": [],
    "userDecryption": {
        "masterPasswordUnlock": { /* KDF 配置 */ }
    },
    "object": "sync"
}
```

### SSH 密钥类型的客户端版本过滤

```rust
// Filter out SSH keys if the client version is less than 2024.12.0
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

客户端版本低于 2024.12.0 时，SSH 密钥类型条目（type=5）在同步中被过滤掉，不返回给客户端。

位置：[ciphers.rs#L128-L137](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L128-L137)

---

## 代码参考

### 核心文件

| 文件 | 作用 |
|------|------|
| [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs) | 个人导入端点、CipherData 结构、字段映射、同步逻辑 |
| [organizations.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs) | 组织导入/导出端点、集合结构、集合关系处理 |
| [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/cipher.rs) | Cipher 数据模型、验证逻辑、序列化导出 |
| [folders.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/folders.rs) | FolderData 结构、文件夹操作 |
| [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/collection.rs) | Collection 数据模型、序列化 |
| [folder.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/folder.rs) | Folder 数据模型、序列化 |
| [error.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/error.rs) | 错误类型、宏定义、API 错误响应序列化 |
| [util.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/util.rs) | JSON Key 大小写转换、反序列化辅助函数 |
| [config.rs](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/config.rs) | 配置项（note 大小限制等） |

### 核心函数速查

| 函数 | 位置 | 作用 |
|------|------|------|
| `post_ciphers_import` | [ciphers.rs#L595-L644](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L595-L644) | 个人导入入口 |
| `post_org_import` | [organizations.rs#L1782-L1866](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L1782-L1866) | 组织导入入口 |
| `update_cipher_from_data` | [ciphers.rs#L395-L576](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L395-L576) | CipherData → Cipher 映射写入 |
| `Cipher::validate_cipher_data` | [cipher.rs#L97-L140](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/cipher.rs#L97-L140) | 导入前置验证 |
| `Cipher::to_json` | [cipher.rs#L145-L412](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/db/models/cipher.rs#L145-L412) | Cipher 序列化导出 |
| `get_org_export` | [organizations.rs#L3101-L3111](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/organizations.rs#L3101-L3111) | 组织导出入口 |
| `convert_json_key_lcase_first` | [util.rs#L739-L778](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/util.rs#L739-L778) | 导出时 JSON Key 大小写转换 |
| `sync` | [ciphers.rs#L121-L204](file:///d:/fz/0601/solo-dogfeeding/code/132-vaultwarden/src/api/core/ciphers.rs#L121-L204) | 个人完整数据同步 |
