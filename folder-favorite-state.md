# 个人文件夹和收藏状态代码实现梳理

## 一、用户私有组织方式

### 1.1 文件夹（Folder）数据模型

**代码路径**: [src/db/models/folder.rs](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs)

文件夹结构定义（L18-L27）：
```rust
pub struct Folder {
    pub uuid: FolderId,           // 文件夹唯一ID
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,
    pub user_uuid: UserId,        // 所属用户ID（私有归属）
    pub name: String,             // 文件夹名称
}
```

**核心特性**：
- 文件夹是**用户私有**的，通过 `user_uuid` 字段与用户绑定
- 每个用户只能访问自己的文件夹（L128-L137 `find_by_uuid_and_user`）
- 文件夹查询严格按用户过滤（L139-L144 `find_by_user`）

### 1.2 文件夹-密码关联（FolderCipher）

**代码路径**: [src/db/models/folder.rs#L29-L35](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L29-L35)

关联表结构：
```rust
pub struct FolderCipher {
    pub cipher_uuid: CipherId,    // 密码条目ID
    pub folder_uuid: FolderId,    // 文件夹ID
}
```

**关联查询优化**：
- L228-L238 `find_by_user()`：同步时一次性查询用户所有密码的文件夹归属
- 返回 `Vec<(CipherId, FolderId)>`，用于构建 HashMap 避免 N+1 查询

### 1.3 文件夹API接口

**代码路径**: [src/api/core/folders.rs](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/folders.rs)

主要端点：
- `GET /folders` - 获取当前用户所有文件夹
- `GET /folders/<id>` - 获取单个文件夹详情
- `POST /folders` - 创建文件夹
- `PUT /folders/<id>` - 更新文件夹
- `DELETE /folders/<id>` - 删除文件夹

---

## 二、收藏状态（Favorite）

### 2.1 收藏数据模型

**代码路径**: [src/db/models/favorite.rs](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs)

收藏结构定义（L11-L17）：
```rust
pub struct Favorite {
    pub user_uuid: UserId,        // 用户ID
    pub cipher_uuid: CipherId,    // 密码条目ID
}
```

**复合主键**：`(user_uuid, cipher_uuid)` - 每个用户对每个密码只能收藏一次

### 2.2 核心方法

- **L21-L31 `is_favorite()`**：检查指定密码是否为用户收藏
- **L34-L68 `set_favorite()`**：设置/取消收藏，自动处理状态变更
  - 状态变更时调用 `User::update_uuid_revision()` 更新用户修订版本（触发同步）
- **L92-L101 `get_all_cipher_uuid_by_user()`**：同步时一次性获取用户所有收藏的密码ID

---

## 三、同步返回机制

### 3.1 同步入口

**代码路径**: [src/api/core/ciphers.rs#L121-L204](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L121-L204)

`sync()` 函数是客户端数据同步的核心入口：

```rust
#[get("/sync?<data..>")]
async fn sync(data: SyncData, headers: Headers, ...) -> JsonResult {
    // 1. 获取用户可见的所有密码（个人 + 共享）
    let mut ciphers = Cipher::find_by_user_visible(&headers.user.uuid, &conn).await;
    
    // 2. 预加载所有同步数据（避免N+1查询）
    let cipher_sync_data = CipherSyncData::new(&headers.user.uuid, CipherSyncType::User, &conn).await;
    
    // 3. 转换为JSON（包含个人状态）
    let ciphers_json: Vec<_> = ciphers.iter().map(|c| 
        c.to_json(..., Some(&cipher_sync_data), CipherSyncType::User, ...)
    ).collect();
    
    // 4. 返回完整同步数据
    Ok(Json(json!({
        "profile": user_json,
        "folders": folders_json,      // 用户文件夹列表
        "collections": collections_json,
        "ciphers": ciphers_json,      // 密码条目（包含个人状态）
        "sends": sends_json,
        "object": "sync"
    })))
}
```

### 3.2 CipherSyncData 同步数据结构

**代码路径**: [src/api/core/ciphers.rs#L2100-L2214](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L2100-L2214)

**核心结构（L2100-L2110）**：
```rust
pub struct CipherSyncData {
    pub cipher_attachments: HashMap<CipherId, Vec<Attachment>>,    // 密码附件
    pub cipher_folders: HashMap<CipherId, FolderId>,               // 密码->文件夹映射（个人状态）
    pub cipher_favorites: HashSet<CipherId>,                       // 收藏密码集合（个人状态）
    pub cipher_collections: HashMap<CipherId, Vec<CollectionId>>,  // 密码->集合映射
    pub cipher_archives: HashMap<CipherId, NaiveDateTime>,         // 归档状态（个人状态）
    pub members: HashMap<OrganizationId, Membership>,              // 组织成员关系
    // ... 权限相关字段
}
```

**构建过程（L2119-L2142）**：
```rust
impl CipherSyncData {
    pub async fn new(user_id: &UserId, sync_type: CipherSyncType, conn: &DbConn) -> Self {
        match sync_type {
            CipherSyncType::User => {
                // 用户同步：加载所有个人状态
                cipher_folders = FolderCipher::find_by_user(user_id, conn).await.into_iter().collect();
                cipher_favorites = Favorite::get_all_cipher_uuid_by_user(user_id, conn).await.into_iter().collect();
                cipher_archives = Archive::find_by_user(user_id, conn).await.into_iter().collect();
            }
            CipherSyncType::Organization => {
                // 组织同步：不加载个人状态（避免客户端问题）
                cipher_folders = HashMap::new();
                cipher_favorites = HashSet::new();
                cipher_archives = HashMap::new();
            }
        }
        // ... 加载其他共享数据
    }
}
```

### 3.3 密码JSON序列化（注入个人状态）

**代码路径**: [src/db/models/cipher.rs#L373-L399](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L373-L399)

在 `to_json()` 方法中，个人状态被注入到每个密码条目中：

```rust
if sync_type == CipherSyncType::User {
    // 文件夹ID（个人状态）
    json_object["folderId"] = json!(if let Some(cipher_sync_data) = cipher_sync_data {
        cipher_sync_data.cipher_folders.get(&self.uuid).cloned()  // 从预加载数据获取
    } else {
        self.get_folder_uuid(user_uuid, conn).await  // 单独查询
    });
    
    // 收藏状态（个人状态）
    json_object["favorite"] = json!(if let Some(cipher_sync_data) = cipher_sync_data {
        cipher_sync_data.cipher_favorites.contains(&self.uuid)  // 从预加载数据获取
    } else {
        self.is_favorite(user_uuid, conn).await  // 单独查询
    });
    
    // 归档状态（个人状态）
    json_object["archivedDate"] = json!(...);
    
    // 权限状态（共享条目的个人权限）
    json_object["edit"] = json!(!read_only);
    json_object["viewPassword"] = json!(!hide_passwords);
}
```

---

## 四、共享条目的个人状态

### 4.1 状态隔离机制

**核心原则**：共享密码条目（组织密码）的**个人状态完全独立存储**，不影响其他用户。

| 状态类型 | 存储表 | 关联键 | 说明 |
|---------|--------|--------|------|
| 文件夹归属 | `folders_ciphers` | `user_uuid` 通过文件夹间接关联 | 同一共享密码，不同用户可放入不同文件夹 |
| 收藏状态 | `favorites` | `(user_uuid, cipher_uuid)` | 同一共享密码，每个用户可独立收藏 |
| 归档状态 | `archives` | `(user_uuid, cipher_uuid)` | 同一共享密码，每个用户可独立归档 |

### 4.2 访问权限限制（个人状态的可见性）

**代码路径**: [src/db/models/cipher.rs#L601-L650](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L601-L650)

`get_access_restrictions()` 方法判断用户对密码的权限：
```rust
pub async fn get_access_restrictions(
    &self,
    user_uuid: &UserId,
    cipher_sync_data: Option<&CipherSyncData>,
    conn: &DbConn,
) -> Option<(bool, bool, bool)> {  // (read_only, hide_passwords, manage)
    
    // 1. 用户自己的密码 - 完全权限
    if self.is_owned_by_user(user_uuid) {
        return Some((false, false, true));
    }
    
    // 2. 组织完全访问权限 - 完全权限
    if self.is_in_full_access_org(user_uuid, cipher_sync_data, conn).await {
        return Some((false, false, true));
    }
    
    // 3. 集合/组权限 - 按配置限制（只读、隐藏密码等）
    // ... 查询集合权限
}
```

**关键特性**：
- 即使是**只读**的共享密码，用户仍可以：
  - 将其移动到自己的私人文件夹（L374-L378 `folderId`）
  - 标记为收藏（L379-L383 `favorite`）
  - 归档到回收站（L384-L388 `archivedDate`）
- 这些操作**只影响当前用户**的视图，不影响其他组织成员

### 4.3 用户修订版本触发

**代码路径**: [src/db/models/favorite.rs#L43, L53](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs#L43-L53)

当个人状态变更时，自动触发用户修订版本更新：
```rust
// 添加收藏时
User::update_uuid_revision(user_uuid, conn).await;

// 取消收藏时
User::update_uuid_revision(user_uuid, conn).await;
```

这会通知所有客户端需要重新同步数据。

---

## 五、关键流程总结

### 5.1 同步数据流程
```
客户端请求 /sync
    ↓
调用 CipherSyncData::new() 预加载所有数据
    ├─ 查询用户所有文件夹归属（FolderCipher）
    ├─ 查询用户所有收藏密码（Favorite）
    ├─ 查询用户所有归档密码（Archive）
    ├─ 查询所有附件
    ├─ 查询集合关联
    └─ 查询权限信息
    ↓
遍历所有可见密码调用 to_json()
    ├─ 注入 folderId（从 cipher_folders HashMap 获取）
    ├─ 注入 favorite（从 cipher_favorites HashSet 获取）
    ├─ 注入 archivedDate（从 cipher_archives HashMap 获取）
    └─ 注入权限信息
    ↓
返回完整同步响应
```

### 5.2 收藏变更流程
```
客户端 POST /ciphers/<id>/favorite
    ↓
调用 Cipher::set_favorite()
    ↓
调用 Favorite::set_favorite()
    ├─ 检查当前状态（避免重复操作）
    ├─ INSERT/DELETE favorites 表
    └─ 调用 User::update_uuid_revision() 触发同步
    ↓
返回更新后的密码（包含最新 favorite 状态）
```

### 5.3 文件夹归属变更流程
```
客户端 PUT /ciphers/<id> （带 folderId）
    ↓
调用 update_cipher_from_data()
    ↓
处理文件夹变更
    ├─ 删除旧的 FolderCipher 关联
    └─ 创建新的 FolderCipher 关联
    ↓
调用 User::update_uuid_revision() 触发同步
    ↓
返回更新后的密码（包含最新 folderId）
```

---

## 六、关键表关系图

```
User (用户)
  │
  ├─ 1:N → Folder (私人文件夹)
  │         └─ M:N → FolderCipher → Cipher (密码条目)
  │
  ├─ M:N → Favorite → Cipher (收藏关联)
  │
  ├─ M:N → Archive → Cipher (归档关联)
  │
  └─ M:N → Membership → Organization (组织成员)
                  │
                  └─ M:N → Collection (组织集合)
                            └─ M:N → Cipher (共享密码)
```

**注意**：
- `Folder`、`Favorite`、`Archive` 是**用户私有**表
- `Cipher` 可以是用户私有（`user_uuid` 有值）或组织共享（`organization_uuid` 有值）
- 共享密码的个人状态（文件夹、收藏、归档）通过用户ID独立存储，互不影响
