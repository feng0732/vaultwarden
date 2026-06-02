# Cipher 同步与数据边界分析文档

## 概述

本文档分析 Vaultwarden 中加密条目（Cipher）的同步机制、数据边界、保存/更新流程、删除与恢复逻辑，以及版本控制机制。

---

## 1. Cipher 数据模型结构

**文件位置**: [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L31-L62)

### 1.1 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `uuid` | `CipherId` | 主键，全局唯一标识 |
| `created_at` | `NaiveDateTime` | 创建时间 |
| `updated_at` | `NaiveDateTime` | 更新时间（版本控制关键字段） |
| `user_uuid` | `Option<UserId>` | 所属用户 ID（个人 vault） |
| `organization_uuid` | `Option<OrganizationId>` | 所属组织 ID（组织 vault） |
| `key` | `Option<String>` | 加密密钥 |
| `atype` | `i32` | 条目类型：1=Login, 2=SecureNote, 3=Card, 4=Identity, 5=SshKey |
| `name` | `String` | 条目名称 |
| `notes` | `Option<String>` | 备注 |
| `fields` | `Option<String>` | 自定义字段（JSON 序列化） |
| `data` | `String` | 类型特定数据（JSON 序列化） |
| `password_history` | `Option<String>` | 密码历史（JSON 序列化） |
| `deleted_at` | `Option<NaiveDateTime>` | 删除时间（软删除标记） |
| `reprompt` | `Option<i32>` | 是否需要重新验证密码 |

### 1.2 数据所有权规则

- **个人条目**: `user_uuid` 有值，`organization_uuid` 为 `None`
- **组织条目**: `organization_uuid` 有值，`user_uuid` 为 `None`
- 两者互斥，不能同时有值

---

## 2. 保存与更新流程

### 2.1 保存方法 (`save`)

**位置**: [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L442-L474)

```rust
pub async fn save(&mut self, conn: &DbConn) -> EmptyResult {
    self.update_users_revision(conn).await;     // 1. 更新用户修订版本
    self.updated_at = Utc::now().naive_utc();   // 2. 更新时间戳
    
    // 3. 数据库操作（支持 SQLite/MySQL/PostgreSQL）
    db_run! { conn:
        sqlite, mysql {
            // 使用 replace_into，遇到外键约束时回退到 update
        }
        postgresql {
            // 使用 INSERT ... ON CONFLICT DO UPDATE
        }
    }
}
```

**关键点**:
- 每次保存都会更新 `updated_at` 时间戳
- 先更新用户修订版本，再执行数据库操作
- 跨数据库兼容性处理

### 2.2 更新用户修订 (`update_users_revision`)

**位置**: [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L414-L440)

- **个人条目**: 只更新所属用户的修订版本
- **组织条目**: 更新所有有权访问该条目的用户修订版本
  - 直接访问集合的用户
  - 通过组访问集合的用户（如果启用了组功能）

### 2.3 从数据更新 Cipher (`update_cipher_from_data`)

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L395-L576)

#### 核心流程：

1. **个人所有权策略检查**
   - 如果组织启用了 "禁止个人 vault" 策略，阻止创建/修改个人条目

2. **版本冲突检测**（关键！）
   ```rust
   if let Some(dt) = data.last_known_revision_date {
       if cipher.updated_at.signed_duration_since(dt).num_seconds() > 1 {
           err!("The client copy of this cipher is out of date...")
       }
   }
   ```
   - 客户端发送 `last_known_revision_date`（本地版本时间）
   - 服务器比较：如果服务器版本比客户端新超过 1 秒，拒绝更新
   - 防止并发修改导致数据丢失

3. **所有权转换检查**
   - 检查组织 ID 匹配
   - 支持从个人 vault 转移到组织 vault

4. **权限验证**
   - 验证用户对目标组织/集合的写入权限

5. **数据清理与验证**
   - 移除 `response` 字段（客户端生成的冗余字段）
   - 验证备注长度限制
   - 处理附件密钥轮换

6. **字段更新**
   - 更新所有业务字段
   - 调用 `save()` 持久化
   - 更新文件夹关联
   - 更新收藏状态
   - 更新归档状态

7. **通知与事件日志**
   - 发送推送通知
   - 记录组织事件日志

### 2.4 版本控制机制总结

| 机制 | 说明 |
|------|------|
| 乐观锁 | 基于 `updated_at` 时间戳 |
| 冲突检测 | 客户端发送 `last_known_revision_date`，服务器比较 |
| 修订传播 | 通过 `update_users_revision` 通知所有相关用户 |
| 推送通知 | 通过 WebSocket 实时同步 |

### 2.5 条目版本时间变化时机详解

**核心原则**：只有调用 `cipher.save()` 才会更新 `cipher.updated_at`（条目本身版本）。

#### ✅ 改变条目版本的操作（调用 `save()`）

| 操作 | 说明 | 代码位置 |
|------|------|---------|
| **创建新条目** | `Cipher::new()` + `save()` | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L361-L375) |
| **更新条目内容** | 修改 name/notes/fields/data 等字段 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L680-L706) |
| **软删除** | 设置 `deleted_at` 字段 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1785-L1787) |
| **从回收站恢复** | 清除 `deleted_at` 字段 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1872-L1873) |
| **分享到组织** | 转换所有权 + 更新字段 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1028-L1077) |
| **添加/删除附件** | 触发条目版本更新 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1317-L1325) |

**`save()` 方法内部实现** ([src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L442-L474)):
```rust
pub async fn save(&mut self, conn: &DbConn) -> EmptyResult {
    self.update_users_revision(conn).await;     // 更新用户修订
    self.updated_at = Utc::now().naive_utc();   // ⬅️ 条目版本时间戳更新
    // 数据库操作...
}
```

#### ❌ 不改变条目版本的操作（只更新用户修订）

这些操作只调用 `User::update_uuid_revision()`，**不**调用 `cipher.save()`，因此：
- `cipher.updated_at` **不变**（条目版本不变）
- `user.updated_at` **变化**（用户同步修订变化）

| 操作 | 说明 | 代码位置 |
|------|------|---------|
| **移动到文件夹** | 修改 FolderCipher 关联表 | [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L518-L551) |
| **收藏/取消收藏** | 修改 favorites 关联表 | [src/db/models/favorite.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/favorite.rs#L34-L68) |
| **归档/取消归档** | 修改 archives 关联表 | [src/db/models/archive.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/archive.rs#L36-L81) |
| **创建/更新文件夹** | 文件夹本身的操作 | [src/db/models/folder.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/folder.rs#L75-L107) |
| **删除文件夹** | 删除文件夹本身 | [src/db/models/folder.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/folder.rs#L109-L119) |
| **批量移动条目** | 多个条目移动文件夹 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1593-L1645) |

**用户修订更新实现** ([src/db/models/user.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/user.rs#L351-L386)):
```rust
pub async fn update_uuid_revision(uuid: &UserId, conn: &DbConn) {
    // 只更新 users.updated_at，不影响 ciphers 表
    Self::update_revision_impl(uuid, &Utc::now().naive_utc(), conn).await;
}
```

#### 📊 两种版本变化的区别

| 维度 | 条目版本 (`cipher.updated_at`) | 用户修订 (`user.updated_at`) |
|------|-------------------------------|-----------------------------|
| **触发条件** | 调用 `cipher.save()` | 调用 `User::update_uuid_revision()` |
| **影响范围** | 所有能看到该条目的用户 | 仅单个用户 |
| **同步信号** | `SyncCipherUpdate` (精确推送) | `SyncCiphers` / `SyncVault` (广播) |
| **客户端行为** | 需要合并/刷新单个条目 | 需要重新同步整个 vault |
| **版本冲突检测** | ✅ 用于乐观锁检测 | ❌ 不用于冲突检测 |

---

## 3. 同步数据结构 (`CipherSyncData`)

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L2096-L2214)

### 3.1 设计目标

解决 **N+1 查询问题**：在全量同步时，通过少数几个批量查询预先加载所有关联数据，避免对每个 cipher 单独查询数据库。

### 3.2 数据结构

```rust
pub struct CipherSyncData {
    // Cipher 关联数据
    pub cipher_attachments: HashMap<CipherId, Vec<Attachment>>,     // 附件
    pub cipher_folders: HashMap<CipherId, FolderId>,               // 文件夹
    pub cipher_favorites: HashSet<CipherId>,                        // 收藏
    pub cipher_collections: HashMap<CipherId, Vec<CollectionId>>,  // 集合
    pub cipher_archives: HashMap<CipherId, NaiveDateTime>,         // 归档
    
    // 权限相关数据
    pub members: HashMap<OrganizationId, Membership>,                          // 组织成员关系
    pub user_collections: HashMap<CollectionId, CollectionUser>,               // 用户-集合权限
    pub user_collections_groups: HashMap<CollectionId, CollectionGroup>,       // 组-集合权限
    pub user_group_full_access_for_organizations: HashSet<OrganizationId>,     // 组完全访问的组织
}
```

### 3.3 同步类型 (`CipherSyncType`)

```rust
pub enum CipherSyncType {
    User,          // 用户同步：包含文件夹、收藏、归档
    Organization,  // 组织同步：不包含用户个人数据
}
```

#### 🔄 个人同步 vs 组织同步：用户侧状态返回差异

**核心原则**：用户侧状态（文件夹、收藏、归档）是**用户级**属性，不是 cipher 本身的属性。

| 用户侧状态 | 个人同步 (`User`) | 组织同步 (`Organization`) | 存储位置 |
|-----------|-------------------|-------------------------|---------|
| **`folderId`** | ✅ 返回（用户所在文件夹） | ❌ 不返回（始终 null） | `folders_ciphers` 表 |
| **`favorite`** | ✅ 返回（是否收藏） | ❌ 不返回（始终 false） | `favorites` 表 |
| **`archivedDate`** | ✅ 返回（归档时间） | ❌ 不返回（始终 null） | `archives` 表 |
| **`edit`** | ✅ 返回（可编辑权限） | ❌ 不返回 | 权限计算 |
| **`viewPassword`** | ✅ 返回（密码可见性） | ❌ 不返回 | 权限计算 |
| **`permissions`** | ✅ 返回（删除/恢复权限） | ❌ 不返回 | 权限计算 |

#### CipherSyncData 数据加载差异 ([src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L2119-L2142))

```rust
match sync_type {
    // User 同步：批量加载用户个人数据
    CipherSyncType::User => {
        cipher_folders = FolderCipher::find_by_user(user_id, conn).await.into_iter().collect();
        cipher_favorites = Favorite::get_all_cipher_uuid_by_user(user_id, conn).await.into_iter().collect();
        cipher_archives = Archive::find_by_user(user_id, conn).await.into_iter().collect();
    }
    // Organization 同步：跳过用户个人数据
    CipherSyncType::Organization => {
        cipher_folders = HashMap::with_capacity(0);    // 空
        cipher_favorites = HashSet::with_capacity(0);  // 空
        cipher_archives = HashMap::with_capacity(0);   // 空
    }
}
```

#### to_json 输出字段差异 ([src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L373-L399))

```rust
// 这些字段仅在 User 同步类型时返回
if sync_type == CipherSyncType::User {
    json_object["folderId"] = ...;      // 文件夹 ID
    json_object["favorite"] = ...;      // 是否收藏
    json_object["archivedDate"] = ...;  // 归档时间
    json_object["edit"] = ...;          // 可编辑
    json_object["viewPassword"] = ...;  // 密码可见
    json_object["permissions"] = ...;   // 权限对象
}
```

#### 🌐 两种同步的使用场景

| 场景 | 使用同步类型 | API 端点 |
|------|-------------|---------|
| **用户登录后全量同步** | `User` | `GET /sync` |
| **用户获取个人 vault 列表** | `User` | `GET /ciphers` |
| **组织管理员查看组织 vault** | `Organization` | `GET /organizations/<id>/details` |
| **组织导出/批量操作** | `Organization` | 内部调用 |

**设计意图**：
- 组织 vault 是共享的，不应该包含任何用户个人的分类信息
- 避免 web-vault 在组织管理界面显示个人文件夹/收藏等无关信息
- 减少不必要的数据传输

### 3.4 同步接口 (`sync`)

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L121-L204)

#### 同步流程：

1. 获取用户基本信息
2. 查询用户可见的所有 cipher
3. 创建 `CipherSyncData`（批量加载所有关联数据）
4. 遍历 cipher，使用预加载的数据生成 JSON
5. 返回完整的同步数据

**返回数据包含**:
- `profile`: 用户信息
- `folders`: 文件夹列表
- `collections`: 集合列表
- `policies`: 组织策略
- `ciphers`: 加密条目列表（核心）
- `domains`: 等效域名
- `sends`: Send 列表
- `userDecryption`: 用户解密选项

---

## 4. 删除与恢复逻辑

### 4.1 删除选项 (`CipherDeleteOptions`)

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1762-L1768)

| 选项 | HTTP 方法 | 说明 |
|------|-----------|------|
| `SoftSingle` | PUT | 软删除单个（移到回收站） |
| `SoftMulti` | PUT | 软删除批量 |
| `HardSingle` | DELETE/POST | 硬删除单个（永久删除） |
| `HardMulti` | DELETE/POST | 硬删除批量 |

### 4.2 软删除流程 (`delete_cipher_by_uuid`)

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1770-L1828)

```rust
if *delete_options == SoftSingle || *delete_options == SoftMulti {
    cipher.deleted_at = Some(Utc::now().naive_utc());  // 标记删除时间
    cipher.save(conn).await?;                          // 保存（更新版本）
    // 发送 SyncCipherUpdate 通知
}
```

**关键点**:
- 只是设置 `deleted_at` 字段，不从数据库删除记录
- 仍然会更新用户修订版本
- 同步时会包含这些条目（客户端根据 `deletedDate` 判断是否显示在回收站）

### 4.3 硬删除流程

```rust
} else {
    cipher.delete(conn).await?;  // 从数据库永久删除
    // 发送 SyncLoginDelete 通知
}
```

**硬删除级联操作** ([src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L476-L490)):
1. 删除所有文件夹关联 (`FolderCipher`)
2. 删除所有集合关联 (`CollectionCipher`)
3. 删除所有附件 (`Attachment`)
4. 删除所有收藏关联 (`Favorite`)
5. 最后删除 cipher 本身

### 4.4 自动清空回收站

**位置**: [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L508-L516)

- 配置项：`trash_auto_delete_days`
- 定时任务：`purge_trashed_ciphers`
- 逻辑：删除 `deleted_at` 早于指定天数的 cipher

### 4.5 恢复流程 (`restore_cipher_by_uuid`)

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1857-L1901)

```rust
cipher.deleted_at = None;  // 清除删除标记
cipher.save(conn).await?;  // 保存（更新版本）
// 发送 SyncCipherUpdate 通知
```

**关键点**:
- 只是将 `deleted_at` 设为 `None`
- 需要写入权限
- 会记录 `CipherRestored` 事件

### 4.6 批量操作特性

- **单条操作**: 发送针对该 cipher 的详细推送通知
- **批量操作**: 只发送通用 `SyncCiphers` 通知，让客户端重新同步
  - 避免大量推送通知风暴
  - 移动、删除、恢复、归档等操作都遵循此规则

---

## 5. 归档与取消归档

### 5.1 归档特性

**位置**: [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1980-L2042)

- 归档是**用户级**属性（不是 cipher 本身的属性）
- 存储在独立的 `archives` 表中，通过 `user_uuid + cipher_uuid` 联合主键
- 同一 cipher 不同用户可以有不同的归档状态
- 只需要读权限就可以归档/取消归档

### 5.2 与删除的区别

| 特性 | 删除 (Soft) | 归档 |
|------|-------------|------|
| 存储位置 | cipher.deleted_at | archives 表 |
| 作用范围 | 全局（所有用户） | 用户个人 |
| 需要权限 | 写入 | 读取 |
| 自动清理 | 是（配置天数后） | 否 |

---

## 6. 数据边界与权限控制

### 6.1 访问限制获取 (`get_access_restrictions`)

**位置**: [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L601-L669)

#### 权限层级（从高到低）：

1. **直接拥有者** (`is_owned_by_user`)
   - `user_uuid` 匹配当前用户
   - 完全权限：`(read_only=false, hide_passwords=false, manage=true)`

2. **组织完全访问** (`is_in_full_access_org`)
   - 成员的 `access_all=true` 或 成员类型是 Owner/Admin
   - 完全权限

3. **组完全访问** (`is_in_full_access_group`)
   - 用户所在组有 `access_all=true`
   - 完全权限

4. **集合用户权限** (`get_user_collections_access_flags`)
   - 直接分配给用户的集合权限
   - 优先级高于组权限

5. **集合组权限** (`get_group_collections_access_flags`)
   - 通过组继承的集合权限

#### 多集合权限合并规则：

```rust
let mut read_only = true;
let mut hide_passwords = true;
let mut manage = false;
for (ro, hp, mn) in &rows {
    read_only &= ro;        // AND：所有集合都是只读才只读
    hide_passwords &= hp;   // AND：所有集合都隐藏密码才隐藏
    manage |= mn;           // OR：任一集合有管理权限就有
}
```

**设计意图**:
- 只要有一个集合可写，整体就可写
- 只要有一个集合能看密码，整体就能看
- 只要有一个集合能管理，整体就能管理

### 6.2 可写性判断 (`is_write_accessible_to_user`)

```rust
match self.get_access_restrictions(...) {
    Some((read_only, _, manage)) => !read_only || manage,
    None => false,
}
```

- `!read_only`: 有写入权限
- `|| manage`: 或有管理权限

### 6.3 可访问性判断 (`is_accessible_to_user`)

```rust
self.get_access_restrictions(...).await.is_some()
```

- 只要能获取到权限元组，就表示有访问权

### 6.4 集合可编辑判断 (`is_in_editable_collection_by_user`)

```rust
Some((read_only, hide_passwords, manage)) => 
    (!read_only && !hide_passwords) || manage,
```

- 用于检查是否可以修改集合关联
- 要求：可写 **且** 密码可见 **或** 有管理权限
- 防止权限提升：不能通过移动集合来绕过密码隐藏限制

---

## 7. 同步返回数据结构 (`to_json`)

**位置**: [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L145-L412)

### 7.1 核心字段

```json
{
  "object": "cipherDetails",
  "id": "uuid",
  "type": 1,
  "creationDate": "ISO时间",
  "revisionDate": "ISO时间",  // 对应 cipher.updated_at（条目版本）
  "deletedDate": "ISO时间或null",
  "reprompt": 0,
  "organizationId": "uuid或null",
  "key": "加密密钥或null",
  "attachments": [...],
  "collectionIds": [...],
  "name": "名称",
  "notes": "备注或null",
  "fields": [...],
  "data": {...},
  "passwordHistory": [...],
  "login/secureNote/card/identity/sshKey": {...},
  
  // ⬇️ User 同步类型特有字段（用户侧状态） ⬇️
  "folderId": "uuid或null",
  "favorite": true/false,
  "archivedDate": "ISO时间或null",
  "edit": true/false,
  "viewPassword": true/false,
  "permissions": {
    "delete": true/false,
    "restore": true/false
  }
}
```

### 7.3 版本字段说明

| 字段 | 对应数据 | 含义 | 变化时机 |
|------|---------|------|---------|
| `creationDate` | `cipher.created_at` | 条目创建时间 | 永不变化 |
| `revisionDate` | `cipher.updated_at` | 条目的**内容版本** | 调用 `cipher.save()` 时变化 |
| `deletedDate` | `cipher.deleted_at` | 软删除时间 | 软删除/恢复时变化 |
| `archivedDate` | `archive.archived_at` | 用户个人归档时间 | 归档/取消归档时变化（仅个人同步） |

**重要提示**：
- `revisionDate` 是条目本身的版本，用于乐观锁冲突检测
- 用户侧操作（移动文件夹、收藏、归档）不会改变 `revisionDate`
- 但这些操作会更新用户的 `user.updated_at`，触发重新同步

### 7.2 数据兼容性处理

- **字段类型修正**: 确保 `type` 是数字（不是字符串）
- **URI 匹配值修正**: 确保 `match` 是数字或 null
- **密码历史清理**: 过滤 null 值的密码记录
- **日期格式标准化**: 统一为 ISO 8601 格式
- **SSH Key 验证**: 缺少必填字段时设为 null（避免客户端崩溃）
- **向后兼容**: 始终提供 `uri` 字段（取第一个 URI）

---

## 8. 关键数据流图示

### 8.1 Cipher 更新流程

```
客户端请求 (带 last_known_revision_date)
         │
         ▼
  版本冲突检测
    ├─ 过期 → 返回错误，要求重新同步
    └─ 有效 → 继续
         │
         ▼
  权限检查 (is_write_accessible_to_user)
    ├─ 无权限 → 返回错误
    └─ 有权限 → 继续
         │
         ▼
  更新 cipher 字段
         │
         ▼
  save() 方法
    ├─ update_users_revision()
    │   └─ 更新所有相关用户的修订版本
    └─ 更新 updated_at 时间戳
         │
         ▼
  发送推送通知 (SyncCipherUpdate)
         │
         ▼
  返回更新后的 cipher JSON
```

### 8.2 全量同步流程

```
客户端 /sync 请求
         │
         ▼
  Cipher::find_by_user_visible()
    → 获取用户可见的所有 cipher
         │
         ▼
  CipherSyncData::new()
    ├─ 批量查询附件
    ├─ 批量查询文件夹
    ├─ 批量查询收藏
    ├─ 批量查询集合
    ├─ 批量查询归档
    ├─ 批量查询成员关系
    └─ 批量查询权限
         │
         ▼
  遍历 cipher，调用 to_json()
    → 使用 CipherSyncData 中的缓存数据
         │
         ▼
  返回完整同步数据
```

---

## 9. 关键代码引用汇总

| 功能 | 文件位置 |
|------|---------|
| Cipher 数据模型 | [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L31-L62) |
| Cipher 保存方法 | [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L442-L474) |
| 更新用户修订 | [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L414-L440) |
| 从数据更新 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L395-L576) |
| 同步接口 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L121-L204) |
| CipherSyncData 结构 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L2096-L2214) |
| 删除逻辑 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1770-L1828) |
| 恢复逻辑 | [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/api/core/ciphers.rs#L1857-L1901) |
| 权限控制 | [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L601-L669) |
| 转 JSON 输出 | [src/db/models/cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/1-vaultwarden/src/db/models/cipher.rs#L145-L412) |
