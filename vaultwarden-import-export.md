# Vaultwarden Bitwarden 导入导出格式兼容性分析

## 目录

1. [概述](#概述)
2. [后端仅支持 Bitwarden JSON 格式](#后端仅支持-bitwarden-json-格式)
3. [格式识别与导入端点](#格式识别与导入端点)
4. [字段映射关系](#字段映射关系)
5. [错误处理机制](#错误处理机制)
6. [数据写入边界与约束](#数据写入边界与约束)
7. [导出格式](#导出格式)
8. [代码参考](#代码参考)

---

## 概述

Vaultwarden 作为 Bitwarden 的兼容服务端实现，提供了完整的个人保险箱和组织保险箱导入导出功能。导入导出基于 Bitwarden 官方 JSON 格式，涉及 Cipher（密码条目）、Folder（文件夹）、Collection（集合）及其关联关系。

---

## 后端仅支持 Bitwarden JSON 格式

### 服务端解析格式的唯一性

**Vaultwarden 服务端仅接受 Bitwarden 标准 JSON 格式的导入数据**，不提供 CSV、LastPass、1Password、KeePass 等其他密码管理器格式的解析逻辑。

#### 证据

对整个 Rust 源码进行全局搜索：

| 搜索关键词 | 匹配文件数 | 结论 |
|-----------|----------|------|
| `csv` / `Csv` / `CSV` | 0 个文件 | 服务端无任何 CSV 解析代码 |
| `format.*import` / `import.*format` | 0 个文件 | 无多格式导入分派逻辑 |
| `lastpass` / `1password` / `keepass` | 0 个文件 | 无非 Bitwarden 格式的导入适配器 |

相关源码搜索范围：`src/` 目录下所有 `*.rs` 文件。

### 客户端-服务端分工

| 角色 | 职责 |
|------|------|
| **客户端（Web Vault / 桌面 / 移动端）** | 解析 CSV、LastPass、1Password、KeePass 等格式 → 在本地转换为 Bitwarden JSON 结构 → 调用后端导入 API |
| **Vaultwarden 服务端** | 仅接收已转换好的 Bitwarden JSON → 验证 → 写入数据库 |

这与 Bitwarden 官方服务端的架构完全一致：导入格式的识别和转换是客户端的职责，服务端只负责处理规范化后的 Bitwarden JSON。

### 服务端接收的数据结构

两个导入端点均通过 `Json<ImportData>` 直接反序列化 JSON 请求体，没有任何格式检测或多格式分派：

- 个人导入：`data: Json<ImportData>` → 定义于 [src/api/core/ciphers.rs#L596](src/api/core/ciphers.rs#L595-L596)
- 组织导入：`data: Json<ImportData>` → 定义于 [src/api/core/organizations.rs#L1785](src/api/core/organizations.rs#L1783-L1786)

---

## 格式识别与导入端点

### 两个核心导入端点

Vaultwarden 支持两种导入场景，分别对应两个独立的 API 端点：

| 端点 | 用途 | 代码位置 |
|------|------|----------|
| `POST /ciphers/import` | 个人保险箱导入 | [src/api/core/ciphers.rs#L595-L644](src/api/core/ciphers.rs#L595-L644) |
| `POST /ciphers/import-organization` | 组织保险箱导入 | [src/api/core/organizations.rs#L1782-L1866](src/api/core/organizations.rs#L1782-L1866) |

### 导入数据结构

#### 个人导入结构 (`ImportData`)

```rust
struct ImportData {
    ciphers: Vec<CipherData>,       // 密码条目列表
    folders: Vec<FolderData>,       // 文件夹列表
    folder_relationships: Vec<RelationsData>,  // 条目-文件夹关系
}
```
位置：[src/api/core/ciphers.rs#L578-L594](src/api/core/ciphers.rs#L578-L594)

#### 组织导入结构 (`ImportData`)

```rust
struct ImportData {
    ciphers: Vec<CipherData>,                   // 密码条目列表
    collections: Vec<FullCollectionData>,       // 集合列表
    collection_relationships: Vec<RelationsData>, // 条目-集合关系
}
```
位置：[src/api/core/organizations.rs#L1764-L1780](src/api/core/organizations.rs#L1764-L1780)

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

映射代码位置：[src/api/core/ciphers.rs#L505-L524](src/api/core/ciphers.rs#L505-L524)

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

映射核心代码：[src/api/core/ciphers.rs#L395-L576](src/api/core/ciphers.rs#L395-L576)

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
位置：[src/api/core/ciphers.rs#L249-L301](src/api/core/ciphers.rs#L249-L301)

**字段反序列化兼容处理**：

1. **`folder_id` 的空字符串兼容**：使用 `deser_opt_nonempty_str` 反序列化器，将空字符串 `""` 转换为 `None`，避免无效的空 ID。
   位置：[src/util.rs#L630-L643](src/util.rs#L630-L643)

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

清洗代码位置：[src/api/core/ciphers.rs#L409-L416](src/api/core/ciphers.rs#L409-L416)

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
位置：[src/api/core/folders.rs#L39-L45](src/api/core/folders.rs#L39-L45)

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
位置：[src/api/core/organizations.rs#L128-L136](src/api/core/organizations.rs#L128-L136)

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
位置：[src/api/core/ciphers.rs#L449-L471](src/api/core/ciphers.rs#L449-L471)

---

## 错误处理机制

导入流程的错误处理分 **两个阶段**，处理方式有本质区别：
- **第一阶段：前置验证** — 在任何数据写入前执行，严格拦截所有不合法的 Cipher
- **第二阶段：逐项写入** — 通过前置验证后，逐条写入数据库，此时出错不再"全有或全无"

### 第一阶段：前置验证（写入前拦截）

在写入任何数据之前，调用 `Cipher::validate_cipher_data()` 对所有 Cipher 做批量检查。此阶段的检查是"全有或全无"——任何一项不合法则整体中止，此时还没有任何数据写入数据库。

```rust
Cipher::validate_cipher_data(&data.ciphers)?;
```
位置：[src/api/core/ciphers.rs#L605](src/api/core/ciphers.rs#L605)
和 [src/api/core/organizations.rs#L1800](src/api/core/organizations.rs#L1800)

#### validate_cipher_data 的检查范围

**`validate_cipher_data()` 仅拦截两类错误：**

##### 1. Notes 字段长度超限

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

位置：[src/db/models/cipher.rs#L97-L109](src/db/models/cipher.rs#L97-L109)
和 [src/config.rs#L782-L786](src/config.rs#L782-L786)

##### 2. 密码历史 password 值为 null（非字符串）

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

位置：[src/db/models/cipher.rs#L111-L127](src/db/models/cipher.rs#L111-L127)

#### validate_cipher_data 不检查的内容

以下错误在前置验证阶段不会被拦截，只能在写入阶段被发现：
- Cipher 类型无效（非 1-5）
- 类型数据缺失
- 权限错误（组织不匹配、无法管理集合等）
- 其他运行时错误

### 前置验证失败响应格式

验证失败时返回结构化错误 JSON（Bitwarden 标准格式），此时未写入任何数据：

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

### 第二阶段：写入过程中的错误处理

通过前置验证后，数据开始逐项写入数据库。此阶段 **不存在全有或全无的事务保证**，错误处理方式在个人导入和组织导入之间有显著差异。

---

## 数据写入边界与约束

### 事务边界

**重要**：Vaultwarden 导入流程 **不在单个数据库事务** 中执行，且没有任何回滚机制。

写入顺序：
1. 前置验证通过后开始写入
2. 文件夹/集合逐个创建或复用（写入数据库）
3. Cipher 逐个写入（写入数据库）
4. 关联关系逐个建立（写入数据库）
5. 用户修订日期更新

如果中途出错，步骤 2 或步骤 3 中已成功写入的数据 **不会被回滚**。

### 个人导入：使用 `?` 中止，但不回滚已写入数据

个人导入在多处使用 `?` 传播错误，一旦出错立即中止整个导入函数：

| 写入环节 | 错误传播方式 | 出错时已写入数据的状态 |
|---------|------------|---------------------|
| 个人所有权策略检查 | `.await?` | 无数据写入（中止在最前面） |
| 文件夹创建循环 | `.await?` | 此前已创建的文件夹保留，不回滚 |
| Cipher 写入循环 | `.await?` | 此前已创建的文件夹和已写入的 Cipher 保留，不回滚 |
| 用户修订更新 | `.await?` | 所有 Cipher 已写入 |

关键代码（Cipher 写入循环）：

```rust
for (index, mut cipher_data) in data.ciphers.into_iter().enumerate() {
    ...
    let mut cipher = Cipher::new(cipher_data.r#type, cipher_data.name.clone());
    update_cipher_from_data(
        &mut cipher, cipher_data, &headers, None, &conn, &nt, UpdateType::None
    ).await?;  // 使用 ? 立即中止，但不回滚已写入的数据
}
```
位置：[src/api/core/ciphers.rs#L631-L637](src/api/core/ciphers.rs#L631-L637)

文件夹创建循环同样使用 `?`：

```rust
for folder in data.folders {
    ...
    let mut new_folder = Folder::new(headers.user.uuid.clone(), folder.name);
    new_folder.save(&conn).await?;  // 出错立即中止，已创建的文件夹不回滚
    ...
}
```
位置：[src/api/core/ciphers.rs#L611-L621](src/api/core/ciphers.rs#L611-L621)

### 组织导入：使用 `.ok()` 忽略单个 Cipher 的写入错误

组织导入在不同环节的错误处理策略不同：

| 写入环节 | 错误传播方式 | 行为 |
|---------|------------|------|
| 组织 ID 检查 | `err!` 宏 | 立即中止 |
| 已有集合权限检查 | `err!` 宏 | 立即中止 |
| 新集合创建权限检查 | `err!` 宏 | 立即中止 |
| 集合创建保存 | `.await?` | 立即中止，已创建的集合不回滚 |
| **Cipher 写入循环** | `.await.ok()` | **忽略单个错误，继续处理后续 Cipher** |
| 集合-条目关联保存 | `.await?` | 立即中止，已保存的关联不回滚 |

关键代码（Cipher 写入循环）：

```rust
for mut cipher_data in data.ciphers {
    cipher_data.folder_id = None;
    let mut cipher = Cipher::new(cipher_data.r#type, cipher_data.name.clone());
    update_cipher_from_data(
        &mut cipher, cipher_data, &headers,
        Some(collections.clone()), &conn, &nt, UpdateType::None,
    )
    .await
    .ok();  // 使用 .ok() 忽略 Result 的 Err 变体，继续下一个 Cipher
    ciphers.push(cipher.uuid);  // 无论成功失败，uuid 都被推入列表
}
```
位置：[src/api/core/organizations.rs#L1839-L1855](src/api/core/organizations.rs#L1839-L1855)

**注意一个边界问题**：即使 `update_cipher_from_data` 返回 Err（Cipher 未成功写入数据库），`cipher.uuid` 仍然被推入 `ciphers` 向量。后续在建立集合-条目关联时：

```rust
for (cipher_index, col_index) in relations {
    let cipher_id = &ciphers[cipher_index];
    let col_id = &collections[col_index];
    CollectionCipher::save(cipher_id, col_id, &conn).await?;
}
```
位置：[src/api/core/organizations.rs#L1858-L1862](src/api/core/organizations.rs#L1858-L1862)

这意味着如果某个 Cipher 写入失败但 uuid 仍在列表中，`CollectionCipher::save` 可能因外键约束而失败，此时会通过 `?` 中止整个导入。

### 写入过程中可能出现的错误类型

#### 权限错误

| 场景 | 错误消息 |
|------|---------|
| 个人所有权策略生效 | `"Due to an Enterprise Policy, you are restricted from saving items to your personal vault."` |
| 无组织添加权限 | `"You don't have permission to add item to organization"` |
| 无集合创建权限 | `"The current user isn't allowed to create new collections"` |
| 无集合管理权限 | `"The current user isn't allowed to manage this collection"` |
| 文件夹不存在 | `"Invalid folder", "Folder does not exist or belongs to another user"` |

#### 数据错误

| 场景 | 错误消息 |
|------|---------|
| Cipher 类型无效（非 1-5） | `"Invalid type"` |
| 类型数据缺失 | `"Data missing"` |
| 组织 ID 不匹配 | `"Organization mismatch. Please resync the client before updating the cipher"` |
| 客户端数据过期 | `"The client copy of this cipher is out of date. Resync the client and try again."` |

#### 导入时的特殊豁免

在导入流程中（`UpdateType::None`），**跳过** 修订日期检查（`last_known_revision_date`），避免因日期不一致导致导入失败：

```rust
if ut != UpdateType::None
    && let Some(dt) = data.last_known_revision_date
{
    // 仅在非导入时检查客户端修订日期是否过期
}
```
位置：[src/api/core/ciphers.rs#L420-L433](src/api/core/ciphers.rs#L420-L433)

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
| Notes 最大长度 | 10,000 / 100,000 | 可配置，默认 10KB，在 validate_cipher_data 中前置拦截 |
| Cipher 类型范围 | 1-5 | Login=1, SecureNote=2, Card=3, Identity=4, SshKey=5，写入阶段检查 |
| 密码历史 password | 必须为字符串 | null 值在 validate_cipher_data 中前置拦截 |
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

位置：[src/api/core/ciphers.rs#L532](src/api/core/ciphers.rs#L532)

### 修订日期更新

导入完成后，会更新用户/组织的修订版本号以触发客户端同步：

- 个人导入：`user.update_revision(&conn)` + `nt.send_user_update(UpdateType::SyncVault, ...)`
- 组织导入：`user.update_revision(&conn)`

---

## 导出格式

### /sync 同步接口 vs 组织导出接口：关键区别

Vaultwarden 有两个与"导出"相关的核心接口，但它们的用途、调用方和返回数据结构有本质区别：

| 对比维度 | `GET /sync` | `GET /organizations/<org_id>/export` |
|---------|-------------|-------------------------------------|
| **设计用途** | 客户端日常同步，保持本地数据与服务端一致 | 专门的组织数据归档导出 |
| **调用方** | 所有 Bitwarden 客户端（浏览器/桌面/移动/CLI） | Web Vault 的组织导出功能 |
| **数据范围** | 用户可见的全部数据（个人 + 所属组织） | 仅指定组织的数据 |
| **权限** | 任何已登录用户 | 仅组织 Admin/Owner |
| **SyncType** | `CipherSyncType::User` | `CipherSyncType::Organization` |
| **文件夹/收藏/归档** | 包含（folders, favorite, archivedDate） | 不包含 |
| **返回顶层字段** | profile, folders, collections, policies, ciphers, domains, sends, userDecryption | collections, ciphers |
| **JSON Key 处理** | 代码手动拼 camelCase + DB 读时 `LowerCase<T>` 自动递归首字母小写 | 对整个 collections 和 ciphers 数组递归做首字母小写转换 |
| **代码位置** | [src/api/core/ciphers.rs#L121-L204](src/api/core/ciphers.rs#L121-L204) | [src/api/core/organizations.rs#L3101-L3111](src/api/core/organizations.rs#L3101-L3111) |

### /sync 同步接口格式（用户侧）

客户端每次打开或定时同步时调用，返回完整的用户视图数据：

```json
{
    "profile": { /* 用户信息 */ },
    "folders": [
        {"id": "...", "revisionDate": "...", "name": "...", "object": "folder"}
    ],
    "collections": [
        {"id": "...", "organizationId": "...", "name": "...", "object": "collection", ...}
    ],
    "policies": [],
    "ciphers": [
        // CipherDetails 格式
    ],
    "domains": { /* 等价域名 */ },
    "sends": [ /* Bitwarden Send */ ],
    "userDecryption": {
        "masterPasswordUnlock": { /* KDF 配置 */ }
    },
    "object": "sync"
}
```

#### /sync 字段命名的两层机制

`/sync` 返回的所有字段均为 **camelCase**（首字母小写，后续单词大写），但并非从某个 PascalCase 源"原样返回"，而是通过以下两层机制产生：

**第一层：顶层字段和模型字段由 Rust 代码手动拼出**

`sync()` 函数在 [src/api/core/ciphers.rs#L191-L203](src/api/core/ciphers.rs#L191-L203) 中通过 `json!({...})` 宏硬编码写出顶层字段：

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

每个子模型的 `to_json()` 同样手动写出 camelCase 字段：

- `Cipher::to_json()` 写出 `"creationDate"`、`"revisionDate"`、`"organizationId"`、`"folderId"`、`"favorite"`、`"archivedDate"` 等 [src/db/models/cipher.rs#L336-L410](src/db/models/cipher.rs#L336-L410)
- `Folder::to_json()` 写出 `"revisionDate"` 等 [src/db/models/folder.rs#L52-L61](src/db/models/folder.rs#L52-L61)
- `Collection::to_json()` 写出 `"externalId"`、`"organizationId"` 等 [src/db/models/collection.rs#L68-L76](src/db/models/collection.rs#L68-L76)

**第二层：数据库存储的 JSON 通过 `LowerCase<T>` 自动递归首字母小写**

Cipher 的 `data`（类型特定数据，如 login/card 的字段）、`fields`（自定义字段）、`password_history`（密码历史）在数据库中以 JSON 字符串形式存储，key 大小写取决于写入时的客户端版本。读取时统一通过 `LowerCase<T>` 反序列化包装器处理：

```rust
// 数据库 data 字段读入时，LowerCase<Value> 自动将所有 key 首字母递归转小写
let mut type_data_json = serde_json::from_str::<LowerCase<Value>>(&self.data)
    .map_or_else(|_| Value::Object(serde_json::Map::new()), |d| d.data);

// fields 和 password_history 同理
let fields_json: Vec<_> = self.fields.as_ref().and_then(|s| {
    serde_json::from_str::<Vec<LowerCase<Value>>>(s)
        .ok()
}) ... ;
```
位置：[src/db/models/cipher.rs#L190-L255](src/db/models/cipher.rs#L190-L255)

`LowerCase<T>` 的实现位于 [src/util.rs#L560-L620](src/util.rs#L560-L620)，其核心访问器 `LowerCaseVisitor::visit_map` 对每个 key 调用 `process_json_key()`：

```rust
while let Some((key, value)) = map.next_entry()? {
    result_map.insert(
        process_json_key(key),          // 当前 key 首字母小写
        convert_json_key_lcase_first(value) // 嵌套 value 递归转换
    );
}
```

因此，**`/sync` 输出不调用 `convert_json_key_lcase_first` 做顶层转换**，而是通过模型代码手动拼字段 + `LowerCase<T>` 自动读取转换这两层机制实现统一的 camelCase 输出。

**关键点**：`CipherSyncType::User` 模式下，每个 Cipher 返回时会附加 `folderId`、`favorite`、`archivedDate`、`edit`、`viewPassword`、`permissions` 等用户专属字段，这些字段也由 `to_json()` 手动拼出为 camelCase。

### 组织导出接口格式

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
        // CipherDetails 格式（不含 folderId/favorite/archivedDate）
    ]
}
```

#### 组织导出的递归首字母小写转换

与 `/sync` 不同，组织导出接口显式调用 `convert_json_key_lcase_first()` 对 **collections 和 ciphers 两个数组整体** 做递归首字母小写转换：

```rust
#[get("/organizations/<org_id>/export")]
async fn get_org_export(org_id: OrganizationId, headers: AdminHeaders, conn: DbConn) -> JsonResult {
    ...
    Ok(Json(json!({
        "collections": convert_json_key_lcase_first(get_org_collections_impl(&org_id, &conn).await),
        "ciphers": convert_json_key_lcase_first(get_org_details_impl(&org_id, &headers.host, &headers.user.uuid, &conn).await?),
    })))
}
```
位置：[src/api/core/organizations.rs#L3101-L3111](src/api/core/organizations.rs#L3101-L3111)

代码注释明确说明了原因：

> `// NOTE: It seems clients can't handle uppercase-first keys!!`
> `//       We need to convert all keys so they have the first character to be a lowercase.`
> `//       Else the export will be just an empty JSON file.`

位置：[src/api/core/organizations.rs#L3095-L3097](src/api/core/organizations.rs#L3095-L3097)

**转换范围**
- `get_org_collections_impl` 调用 `Collection::to_json()`，`get_org_details_impl` 调用 `Cipher::to_json(..., CipherSyncType::Organization)`，这两个函数的返回值整体被传入 `convert_json_key_lcase_first()` 做递归转换
- 顶层的 `"collections"` 和 `"ciphers"` 这两个 key 是 `get_org_export` 直接写出的，不在转换范围内

### CipherSyncType 对导出数据的影响

两种同步类型在 `Cipher::to_json()` 中走不同分支：

```rust
// User Sync 支持 Folders, Favorites, and Archives
// Organization Sync 不支持这些，如果设置会导致 web-vault 问题
match sync_type {
    CipherSyncType::User => {
        cipher_folders = FolderCipher::find_by_user(...);
        cipher_favorites = Favorite::get_all_cipher_uuid_by_user(...);
        cipher_archives = Archive::find_by_user(...);
    }
    CipherSyncType::Organization => {
        cipher_folders = HashMap::with_capacity(0);
        cipher_favorites = HashSet::with_capacity(0);
        cipher_archives = HashMap::with_capacity(0);
    }
}
```
位置：[src/api/core/ciphers.rs#L2123-L2142](src/api/core/ciphers.rs#L2123-L2142)

同时在 `to_json()` 中：

```rust
if sync_type == CipherSyncType::User {
    json_object["folderId"] = ...;
    json_object["favorite"] = ...;
    json_object["archivedDate"] = ...;
    json_object["edit"] = ...;
    json_object["viewPassword"] = ...;
    json_object["permissions"] = ...;
}
// 否则不添加以上字段
```
位置：[src/db/models/cipher.rs#L373-L399](src/db/models/cipher.rs#L373-L399)

### 两种 key 小写转换机制的对比

Vaultwarden 中存在 **两种独立** 的 key 首字母小写转换机制，分别服务于不同场景：

| 机制 | 触发时机 | 作用范围 | 代码位置 |
|------|---------|---------|----------|
| `LowerCase<T>` 反序列化包装器 | 从数据库读取 JSON 字符串时 | `Cipher.data`、`Cipher.fields`、`Cipher.password_history` 的所有嵌套 key | [src/util.rs#L560-L620](src/util.rs#L560-L620) |
| `convert_json_key_lcase_first()` 后处理函数 | 组织导出接口返回前 | collections 和 ciphers 两个数组整体（全递归） | [src/util.rs#L621-L778](src/util.rs#L621-L778) |

#### `process_json_key` 的转换规则

两种机制底层均调用同一个 key 处理函数：

```rust
fn process_json_key(key: &str) -> String {
    match key.to_lowercase().as_ref() {
        "ssn" => "ssn".into(),   // 特殊处理：SSN（社会安全号）保持全小写
        _ => lcase_first(key),   // 其余首字母小写（如 CreationDate → creationDate）
    }
}
```

位置：[src/util.rs#L621-L628](src/util.rs#L621-L628)

`convert_json_key_lcase_first()` 对传入的 JSON 值做递归处理：
- 数组：对每个元素递归调用
- 对象：对每个 key 调用 `process_json_key()`，value 递归调用
- 其他类型：原样返回

位置：[src/util.rs#L739-L778](src/util.rs#L739-L778)

### Cipher 导出格式（CipherDetails）

`Cipher::to_json()` 生成完整的 CipherDetails 响应，包含大量向后兼容处理：

| 兼容项 | 处理方式 |
|--------|---------|
| Login Uri match 类型 | 字符串转为数字，失败则为 null |
| Login uri 字段 | 取 uris[0].uri 作为兼容字段 |
| passwordRevisionDate | 日期格式校验和修正 |
| SecureNote type | 缺失或非数字时，默认设为 `{"type": 0}` |
| SSH Key 字段完整性 | 缺关键字段时 type_data 设为 null |
| Fields type 字段 | 字符串转数字，默认值 1（隐藏类型） |
| Password History | 过滤 null 密码值，修正 lastUsedDate 格式 |

位置：[src/db/models/cipher.rs#L145-L412](src/db/models/cipher.rs#L145-L412)

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
        "uris": [],
        "username": "...",
        "password": "..."
    },
    "passwordHistory": [],
    "login": { /* 类型数据，与 type 匹配 */ },
    "secureNote": null,
    "card": null,
    "identity": null,
    "sshKey": null,
    "folderId": null,      // 仅 User Sync 出现
    "favorite": false,     // 仅 User Sync 出现
    "archivedDate": null,  // 仅 User Sync 出现
    "edit": true,          // 仅 User Sync 出现
    "viewPassword": true,  // 仅 User Sync 出现
    "permissions": {       // 仅 User Sync 出现
        "delete": true,
        "restore": true
    }
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

位置：[src/api/core/ciphers.rs#L128-L137](src/api/core/ciphers.rs#L128-L137)

---

## 代码参考

### 核心文件

| 文件 | 作用 |
|------|------|
| [src/api/core/ciphers.rs](src/api/core/ciphers.rs) | 个人导入端点、CipherData 结构、字段映射、同步逻辑 |
| [src/api/core/organizations.rs](src/api/core/organizations.rs) | 组织导入/导出端点、集合结构、集合关系处理 |
| [src/db/models/cipher.rs](src/db/models/cipher.rs) | Cipher 数据模型、验证逻辑、序列化导出 |
| [src/api/core/folders.rs](src/api/core/folders.rs) | FolderData 结构、文件夹操作 |
| [src/db/models/collection.rs](src/db/models/collection.rs) | Collection 数据模型、序列化 |
| [src/db/models/folder.rs](src/db/models/folder.rs) | Folder 数据模型、序列化 |
| [src/error.rs](src/error.rs) | 错误类型、宏定义、API 错误响应序列化 |
| [src/util.rs](src/util.rs) | JSON Key 大小写转换、反序列化辅助函数 |
| [src/config.rs](src/config.rs) | 配置项（note 大小限制等） |

### 核心函数速查

| 函数/机制 | 位置 | 作用 |
|---------|------|------|
| `post_ciphers_import` | [src/api/core/ciphers.rs#L595-L644](src/api/core/ciphers.rs#L595-L644) | 个人导入入口 |
| `post_org_import` | [src/api/core/organizations.rs#L1782-L1866](src/api/core/organizations.rs#L1782-L1866) | 组织导入入口 |
| `update_cipher_from_data` | [src/api/core/ciphers.rs#L395-L576](src/api/core/ciphers.rs#L395-L576) | CipherData → Cipher 映射写入 |
| `Cipher::validate_cipher_data` | [src/db/models/cipher.rs#L97-L140](src/db/models/cipher.rs#L97-L140) | 导入前置验证 |
| `Cipher::to_json` | [src/db/models/cipher.rs#L145-L412](src/db/models/cipher.rs#L145-L412) | Cipher 序列化导出（手动拼 camelCase 字段） |
| `Folder::to_json` | [src/db/models/folder.rs#L52-L61](src/db/models/folder.rs#L52-L61) | Folder 序列化（手动拼 camelCase） |
| `Collection::to_json` | [src/db/models/collection.rs#L68-L76](src/db/models/collection.rs#L68-L76) | Collection 序列化（手动拼 camelCase） |
| `sync` | [src/api/core/ciphers.rs#L121-L204](src/api/core/ciphers.rs#L121-L204) | 个人完整数据同步（手动拼 camelCase 顶层字段） |
| `get_org_export` | [src/api/core/organizations.rs#L3101-L3111](src/api/core/organizations.rs#L3101-L3111) | 组织导出入口 |
| `get_org_details_impl` | [src/api/core/organizations.rs#L896-L910](src/api/core/organizations.rs#L896-L910) | 组织导出 Cipher 数据获取 |
| `get_org_collections_impl` | [src/api/core/organizations.rs#L491-L493](src/api/core/organizations.rs#L491-L493) | 组织导出 Collection 数据获取 |
| `LowerCase<T>` | [src/util.rs#L560-L620](src/util.rs#L560-L620) | DB JSON 读入时自动递归首字母小写的反序列化包装器 |
| `convert_json_key_lcase_first` | [src/util.rs#L739-L778](src/util.rs#L739-L778) | 组织导出时对整个 JSON 值递归首字母小写转换 |
| `process_json_key` | [src/util.rs#L621-L628](src/util.rs#L621-L628) | 单个 key 首字母小写处理（ssn 特殊全小写） |
| `deser_opt_nonempty_str` | [src/util.rs#L630-L643](src/util.rs#L630-L643) | 反序列化时空字符串转 None（folder_id 等字段） |
