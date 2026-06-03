# 个人文件夹和收藏状态代码实现梳理

## 一、用户私有组织方式

### 1.1 文件夹（Folder）数据模型

**代码路径**: [src/db/models/folder.rs#L18-L27](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L18-L27)

```rust
pub struct Folder {
    pub uuid: FolderId,
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,
    pub user_uuid: UserId,        // 所属用户ID（私有归属）
    pub name: String,
}
```

核心特性：
- 文件夹是**用户私有**的，通过 `user_uuid` 字段与用户绑定
- 每个用户只能访问自己的文件夹（[find_by_uuid_and_user](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L128-L137)）
- 文件夹查询严格按用户过滤（[find_by_user](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L139-L144)）

### 1.2 文件夹-密码关联（FolderCipher）

**代码路径**: [src/db/models/folder.rs#L29-L35](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L29-L35)

```rust
pub struct FolderCipher {
    pub cipher_uuid: CipherId,
    pub folder_uuid: FolderId,
}
```

关联查询优化：
- [find_by_user](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L228-L238)：同步时一次性查询用户所有密码的文件夹归属，返回 `Vec<(CipherId, FolderId)>`，用于构建 HashMap 避免 N+1 查询

### 1.3 文件夹API接口

**代码路径**: [src/api/core/folders.rs](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/folders.rs)

- `GET /folders` - 获取当前用户所有文件夹
- `GET /folders/<id>` - 获取单个文件夹详情
- `POST /folders` - 创建文件夹
- `PUT /folders/<id>` - 更新文件夹
- `DELETE /folders/<id>` - 删除文件夹

---

## 二、收藏状态（Favorite）

### 2.1 收藏数据模型

**代码路径**: [src/db/models/favorite.rs#L11-L17](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs#L11-L17)

```rust
#[diesel(primary_key(user_uuid, cipher_uuid))]
pub struct Favorite {
    pub user_uuid: UserId,
    pub cipher_uuid: CipherId,
}
```

复合主键 `(user_uuid, cipher_uuid)`：每个用户对每个密码只能收藏一次。

### 2.2 核心方法

- [is_favorite](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs#L21-L31)：检查指定密码是否为用户收藏
- [set_favorite](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs#L34-L68)：设置/取消收藏，自动处理状态变更；状态变更时调用 `User::update_uuid_revision()` 触发同步
- [get_all_cipher_uuid_by_user](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs#L92-L101)：同步时一次性获取用户所有收藏的密码ID

---

## 三、只读条目的更新路径

### 3.1 两条更新路径的对比

Vaultwarden 对密码条目的更新提供了**两条路径**，它们对权限的要求有本质区别：

| 路径 | 端点 | 权限要求 | 可更新内容 | 适用场景 |
|------|------|---------|-----------|---------|
| 完整更新 | `PUT /ciphers/<id>` | 需要**写权限** (`is_write_accessible_to_user`) | 条目全部字段 + folderId + favorite | 拥有写权限的条目 |
| 部分更新 | `PUT /ciphers/<id>/partial` | 仅需**可访问** (`is_accessible_to_user`) | 仅 folderId + favorite | 只读共享条目 |

### 3.2 完整更新路径 — put_cipher

**代码路径**: [src/api/core/ciphers.rs#L680-L706](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L680-L706)

```rust
#[put("/ciphers/<cipher_id>", data = "<data>")]
async fn put_cipher(cipher_id: CipherId, data: Json<CipherData>, headers: Headers, conn: DbConn, nt: Notify<'_>) -> JsonResult {
    let Some(mut cipher) = Cipher::find_by_uuid(&cipher_id, &conn).await else {
        err!("Cipher doesn't exist")
    };

    // TODO: Check if only the folder ID or favorite status is being changed.
    // These are per-user properties that technically aren't part of the
    // cipher itself, so the user shouldn't need write access to change these.
    // Interestingly, upstream Bitwarden doesn't properly handle this either.

    if !cipher.is_write_accessible_to_user(&headers.user.uuid, &conn).await {
        err!("Cipher is not write accessible")
    }

    update_cipher_from_data(&mut cipher, data, &headers, None, &conn, &nt, UpdateType::SyncCipherUpdate).await?;
    Ok(Json(cipher.to_json(...).await?))
}
```

**关键点**：
- 要求 `is_write_accessible_to_user()` 为 true（即 `!read_only || manage`）
- 代码注释的 TODO 指出：当只有 folderId 或 favorite 变更时，理论上不应要求写权限——因为这是用户个人属性而非条目本身的属性——但 Bitwarden 上游也未正确处理此情况
- 最终调用 `update_cipher_from_data()`，同时更新条目内容和个人状态

### 3.3 部分更新路径 — put_cipher_partial（只读条目的唯一更新入口）

**代码路径**: [src/api/core/ciphers.rs#L718-L748](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L718-L748)

```rust
// Only update the folder and favorite for the user, since this cipher is read-only
#[put("/ciphers/<cipher_id>/partial", data = "<data>")]
async fn put_cipher_partial(
    cipher_id: CipherId,
    data: Json<PartialCipherData>,
    headers: Headers,
    conn: DbConn,
) -> JsonResult {
    let data: PartialCipherData = data.into_inner();

    let Some(cipher) = Cipher::find_by_uuid(&cipher_id, &conn).await else {
        err!("Cipher does not exist")
    };

    // 权限检查：仅需可访问（而非写权限）
    if !cipher.is_accessible_to_user(&headers.user.uuid, &conn).await {
        err!("Cipher does not exist", "Cipher is not accessible for the current user")
    }

    // 文件夹归属校验
    if let Some(ref folder_id) = data.folder_id
        && Folder::find_by_uuid_and_user(folder_id, &headers.user.uuid, &conn).await.is_none()
    {
        err!("Invalid folder", "Folder does not exist or belongs to another user");
    }

    // Move cipher（修改 FolderCipher 关联）
    cipher.move_to_folder(data.folder_id.clone(), &headers.user.uuid, &conn).await?;
    // Update favorite（修改 favorites 表）
    cipher.set_favorite(Some(data.favorite), &headers.user.uuid, &conn).await?;

    Ok(Json(cipher.to_json(&headers.host, &headers.user.uuid, None, CipherSyncType::User, &conn).await?))
}
```

**核心设计**：
- 注释 `// Only update the folder and favorite for the user, since this cipher is read-only` 明确说明了此端点的设计意图
- **权限降级**：只要求 `is_accessible_to_user()`，不要求写权限。因此对只读共享条目，用户仍可修改自己的文件夹归属和收藏状态
- **不触碰条目本身**：只操作 `folders_ciphers` 和 `favorites` 这两个用户私有关联表，不修改 `ciphers` 表的任何字段
- **不发送 WebSocket 通知**：与完整更新不同，partial 更新不触发 `nt.send_cipher_update()`，仅通过用户修订版本变更间接通知

### 3.4 PartialCipherData 数据结构

**代码路径**: [src/api/core/ciphers.rs#L305-L309](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L305-L309)

```rust
pub struct PartialCipherData {
    #[serde(default, deserialize_with = "deser_opt_nonempty_str")]
    folder_id: Option<FolderId>,   // 目标文件夹ID，None 表示移出文件夹
    favorite: bool,                 // 是否收藏
}
```

注意 `favorite` 是 `bool` 而非 `Option<bool>`——每次调用都必须明确指定收藏状态。

---

## 四、folderId 与 favorite 的写入过程

### 4.1 完整更新中的写入（update_cipher_from_data）

**代码路径**: [src/api/core/ciphers.rs#L395-L576](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L395-L576)

写入顺序：

```
1. enforce_personal_ownership_policy()        — 策略检查
2. 版本冲突检测 (last_known_revision_date)    — 防止覆盖
3. 组织归属变更检查                           — 共享/取消共享
4. 文件夹归属校验 (L473-L477)                 — 验证 folder 属于当前用户
5. 附件处理                                   — 附件键/文件名更新
6. cipher 字段更新 + cipher.save()            — 保存条目本身
7. cipher.move_to_folder() (L535)             — 写入 folders_ciphers 关联
8. cipher.set_favorite() (L536)               — 写入 favorites 关联
9. cipher.set_archived_at() (L538-L543)       — 写入 archives 关联
10. 事件日志 + WebSocket 通知                  — 通知其他客户端
```

**关键细节**：
- 步骤 7-9 都是**用户级别的操作**，与条目本身是否属于组织无关
- 即使条目 `organization_uuid` 有值（共享条目），folderId/favorite/archivedDate 仍然针对当前操作用户写入
- 条目保存（步骤6）和个人状态写入（步骤7-9）**不在同一个事务中**（代码注释 L799-800 提到此事）

### 4.2 move_to_folder 的实现细节

**代码路径**: [src/db/models/cipher.rs#L518-L551](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L518-L551)

```rust
pub async fn move_to_folder(
    &self,
    folder_uuid: Option<FolderId>,
    user_uuid: &UserId,
    conn: &DbConn,
) -> EmptyResult {
    User::update_uuid_revision(user_uuid, conn).await;  // 先触发修订版本更新

    match (self.get_folder_uuid(user_uuid, conn).await, folder_uuid) {
        (None, None) => Ok(()),                                          // 无变化
        (Some(ref old), Some(ref new)) if old == new => Ok(()),         // 无变化
        (None, Some(new_folder)) => {                                    // 添加到文件夹
            FolderCipher::new(new_folder, self.uuid.clone()).save(conn).await
        }
        (Some(old_folder), None) => {                                   // 从文件夹移出
            if let Some(old_fc) = FolderCipher::find_by_folder_and_cipher(&old_folder, &self.uuid, conn).await {
                old_fc.delete(conn).await
            } else {
                err!("Couldn't move from previous folder")
            }
        }
        (Some(old_folder), Some(new_folder)) => {                       // 移动到另一文件夹
            if let Some(old_fc) = FolderCipher::find_by_folder_and_cipher(&old_folder, &self.uuid, conn).await {
                old_fc.delete(conn).await?;
            }
            FolderCipher::new(new_folder, self.uuid.clone()).save(conn).await
        }
    }
}
```

**核心要点**：
- **每个用户对同一密码最多一条 FolderCipher 记录**：`get_folder_uuid()` 通过 `folders.user_uuid` JOIN 过滤，只取当前用户的关联
- 旧关联记录通过 `(folder_uuid, cipher_uuid)` 精确定位并删除
- 新关联直接 INSERT，不检查是否已有其他用户的关联（因为不同用户有不同的文件夹）
- 操作开始前先调用 `User::update_uuid_revision()` 通知客户端需同步

### 4.3 set_favorite 的实现细节

**代码路径**: [src/db/models/cipher.rs#L745-L750](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L745-L750) → [src/db/models/favorite.rs#L34-L68](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/favorite.rs#L34-L68)

```rust
// Cipher 层
pub async fn set_favorite(&self, favorite: Option<bool>, user_uuid: &UserId, conn: &DbConn) -> EmptyResult {
    match favorite {
        None => Ok(()),   // 不变更
        Some(status) => Favorite::set_favorite(status, &self.uuid, user_uuid, conn).await,
    }
}

// Favorite 层
pub async fn set_favorite(favorite: bool, cipher_uuid: &CipherId, user_uuid: &UserId, conn: &DbConn) -> EmptyResult {
    let (old, new) = (Self::is_favorite(cipher_uuid, user_uuid, conn).await, favorite);
    match (old, new) {
        (false, true) => {   // 添加收藏
            User::update_uuid_revision(user_uuid, conn).await;
            diesel::insert_into(favorites::table).values(...).execute(conn)...
        }
        (true, false) => {   // 取消收藏
            User::update_uuid_revision(user_uuid, conn).await;
            diesel::delete(favorites::table.filter(...)).execute(conn)...
        }
        _ => Ok(()),         // 状态未变，不触发修订
    }
}
```

**关键细节**：
- `Cipher::set_favorite()` 接受 `Option<bool>`，`None` 表示不变更（用于 `CipherData.favorite`）
- `Favorite::set_favorite()` 先检查当前状态再决定操作，**避免无意义的修订版本更新**
- 收藏操作**完全不涉及 Cipher 表**的读写，只操作 `favorites` 表

### 4.4 共享（Share）时的个人状态处理

**代码路径**: [src/api/core/ciphers.rs#L1028-L1077](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L1028-L1077)

当用户将个人密码共享给组织时（`share_cipher_by_uuid`）：

1. 检查写权限 → `cipher.is_write_accessible_to_user()`
2. 将密码加入组织的集合 → `CollectionCipher::save()`
3. 调用 `update_cipher_from_data()` → 其中会将 `cipher.user_uuid` 设为 `None`、`cipher.organization_uuid` 设为组织ID
4. **关键**：`update_cipher_from_data()` 仍会执行 `move_to_folder()` 和 `set_favorite()`——这意味着**密码从个人变为组织共享后，原用户的文件夹归属和收藏状态仍然保留**

```rust
// update_cipher_from_data 中（L457-L463）
cipher.organization_uuid = Some(org_id);
cipher.user_uuid = None;  // 用户ID被清除
// 但后续仍执行：
cipher.move_to_folder(data.folder_id, &headers.user.uuid, conn).await?;  // 保留文件夹归属
cipher.set_favorite(data.favorite, &headers.user.uuid, conn).await?;     // 保留收藏状态
```

---

## 五、文件夹归属的校验逻辑

### 5.1 写入时的校验

在所有涉及 folderId 写入的路径中，都有相同的校验逻辑：

**代码路径**：
- [update_cipher_from_data](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L473-L477) (L473-L477)
- [put_cipher_partial](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L736-L740) (L736-L740)
- [post_ciphers_create](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L473-L477) (通过 update_cipher_from_data)
- [share_cipher_by_uuid](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L1603-L1605) (通过 update_cipher_from_data)

校验代码完全一致：
```rust
if let Some(ref folder_id) = data.folder_id
    && Folder::find_by_uuid_and_user(folder_id, &headers.user.uuid, conn).await.is_none()
{
    err!("Invalid folder", "Folder does not exist or belongs to another user");
}
```

**校验逻辑**：
1. 仅当 `folder_id` 为 `Some` 时校验（`None` 表示移出文件夹，不需要校验）
2. 调用 `Folder::find_by_uuid_and_user()` 同时检查文件夹**存在性**和**归属**
3. 该查询内部同时过滤 `folders::uuid` 和 `folders::user_uuid`（[L128-L137](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L128-L137)），确保用户只能将密码放入自己的文件夹

### 5.2 读取时的过滤

在读取密码的文件夹归属时，同样通过用户ID过滤：

**代码路径**: [Cipher::get_folder_uuid](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L764-L775)

```rust
pub async fn get_folder_uuid(&self, user_uuid: &UserId, conn: &DbConn) -> Option<FolderId> {
    folders_ciphers::table
        .inner_join(folders::table)
        .filter(folders::user_uuid.eq(&user_uuid))           // 关键：通过 JOIN folders 过滤用户
        .filter(folders_ciphers::cipher_uuid.eq(&self.uuid))
        .select(folders_ciphers::folder_uuid)
        .first::<FolderId>(conn)
        .ok()
}
```

**核心设计**：`folders_ciphers` 表本身没有 `user_uuid` 字段，用户隔离通过 JOIN `folders` 表实现。同一个密码可以有多个 FolderCipher 记录（来自不同用户），查询时通过 `folders.user_uuid` 过滤出当前用户的那条。

### 5.3 同步时的批量查询

**代码路径**: [FolderCipher::find_by_user](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/folder.rs#L228-L238)

```rust
pub async fn find_by_user(user_uuid: &UserId, conn: &DbConn) -> Vec<(CipherId, FolderId)> {
    folders_ciphers::table
        .inner_join(folders::table)                          // JOIN folders
        .filter(folders::user_uuid.eq(user_uuid))            // 按用户过滤
        .select(folders_ciphers::all_columns)                 // 只取 cipher_uuid, folder_uuid
        .load::<(CipherId, FolderId)>(conn)
        .unwrap_or_default()
}
```

同步时一次性获取用户所有密码-文件夹映射，转为 `HashMap<CipherId, FolderId>` 用于 `to_json()` 中快速查找。

---

## 六、可访问性判断

### 6.1 判断层级总览

```
is_accessible_to_user(user_uuid)              ← 最外层：是否有任何访问权限
  └─ get_access_restrictions(user_uuid)       ← 返回权限详情
       ├─ is_owned_by_user(user_uuid)         ← 第1层：直接所有权
       ├─ is_in_full_access_org(user_uuid)    ← 第2层：组织管理员/所有者
       ├─ is_in_full_access_group(user_uuid)  ← 第3层：组全访问权限
       └─ 集合/组权限查询                      ← 第4层：具体集合权限
```

### 6.2 get_access_restrictions — 核心权限判断

**代码路径**: [src/db/models/cipher.rs#L601-L669](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L601-L669)

```rust
pub async fn get_access_restrictions(
    &self,
    user_uuid: &UserId,
    cipher_sync_data: Option<&CipherSyncData>,
    conn: &DbConn,
) -> Option<(bool, bool, bool)> {  // (read_only, hide_passwords, manage)
    // 第1层：用户直接拥有 → 完全权限
    if self.is_owned_by_user(user_uuid)
        // 第2层：组织中拥有全访问权限（Owner/Admin 或 access_all）→ 完全权限
        || self.is_in_full_access_org(user_uuid, cipher_sync_data, conn).await
        // 第3层：所在组拥有全访问权限 → 完全权限
        || self.is_in_full_access_group(user_uuid, cipher_sync_data, conn).await
    {
        return Some((false, false, true));  // 可编辑、可查看密码、可管理
    }

    // 第4层：查询具体集合权限
    let rows = if let Some(cipher_sync_data) = cipher_sync_data {
        // 从预加载数据获取
        // 遍历密码所在的所有集合，取用户/组权限
    } else {
        // 单独查询数据库
        let user_permissions = self.get_user_collections_access_flags(user_uuid, conn).await;
        if user_permissions.is_empty() {
            self.get_group_collections_access_flags(user_uuid, conn).await
        } else {
            user_permissions  // 用户权限优先于组权限
        }
    };

    if rows.is_empty() {
        return None;  // 无任何访问权限
    }

    // 合并多集合权限：read_only 和 hide_passwords 取 AND，manage 取 OR
    let mut read_only = true;
    let mut hide_passwords = true;
    let mut manage = false;
    for (ro, hp, mn) in &rows {
        read_only &= ro;
        hide_passwords &= hp;
        manage |= mn;
    }
    Some((read_only, hide_passwords, manage))
}
```

**权限合并策略**（L645-L668）：
- **read_only**：所有集合都为只读时才只读（AND 语义）——只要有一个集合允许写，就允许写
- **hide_passwords**：所有集合都隐藏密码时才隐藏（AND 语义）——只要有一个集合可见，就可见
- **manage**：任一集合允许管理即可（OR 语义）
- **用户权限优先于组权限**：当用户直接被分配了集合权限时，组权限不再查询（L632-L637）

### 6.3 is_owned_by_user — 直接所有权

**代码路径**: [src/db/models/cipher.rs#L553-L556](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L553-L556)

```rust
pub fn is_owned_by_user(&self, user_uuid: &UserId) -> bool {
    self.user_uuid.is_some() && self.user_uuid.as_ref().unwrap() == user_uuid
}
```

检查 `cipher.user_uuid` 是否匹配。个人密码的 `user_uuid` 有值，组织密码的 `user_uuid` 为 `None`。

### 6.4 is_in_full_access_org — 组织全访问权限

**代码路径**: [src/db/models/cipher.rs#L558-L575](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L558-L575)

```rust
async fn is_in_full_access_org(&self, user_uuid: &UserId, cipher_sync_data: Option<&CipherSyncData>, conn: &DbConn) -> bool {
    if let Some(ref org_uuid) = self.organization_uuid {
        if let Some(cipher_sync_data) = cipher_sync_data {
            if let Some(cached_member) = cipher_sync_data.members.get(org_uuid) {
                return cached_member.has_full_access();
            }
        } else if let Some(member) = Membership::find_confirmed_by_user_and_org(user_uuid, org_uuid, conn).await {
            return member.has_full_access();
        }
    }
    false
}
```

[has_full_access](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/organization.rs#L818-L820) 的定义：
```rust
pub fn has_full_access(&self) -> bool {
    (self.access_all || self.atype >= MembershipType::Admin) && self.has_status(MembershipStatus::Confirmed)
}
```

条件：`access_all = true`（拥有所有集合访问权限）**或**成员类型 >= Admin（Owner/Admin），且状态为 Confirmed。

### 6.5 is_in_full_access_group — 组全访问权限

**代码路径**: [src/db/models/cipher.rs#L577-L594](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L577-L594)

```rust
async fn is_in_full_access_group(&self, user_uuid: &UserId, cipher_sync_data: Option<&CipherSyncData>, conn: &DbConn) -> bool {
    if !CONFIG.org_groups_enabled() { return false; }
    if let Some(ref org_uuid) = self.organization_uuid {
        if let Some(cipher_sync_data) = cipher_sync_data {
            return cipher_sync_data.user_group_full_access_for_organizations.contains(org_uuid);
        }
        return Group::is_in_full_access_group(user_uuid, org_uuid, conn).await;
    }
    false
}
```

条件：用户所在组设置了 `access_all = true`，即该组可访问组织内所有集合。

### 6.6 is_write_accessible_to_user vs is_accessible_to_user

**代码路径**: [src/db/models/cipher.rs#L719-L737](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L719-L737)

```rust
pub async fn is_write_accessible_to_user(&self, user_uuid: &UserId, conn: &DbConn) -> bool {
    match self.get_access_restrictions(user_uuid, None, conn).await {
        Some((read_only, _hide_passwords, manage)) => !read_only || manage,
        None => false,
    }
}

pub async fn is_accessible_to_user(&self, user_uuid: &UserId, conn: &DbConn) -> bool {
    self.get_access_restrictions(user_uuid, None, conn).await.is_some()
}
```

| 方法 | 含义 | 权限要求 |
|------|------|---------|
| `is_accessible_to_user` | 是否有任何访问权限 | `get_access_restrictions` 返回 `Some(...)` |
| `is_write_accessible_to_user` | 是否可编辑 | `!read_only \|\| manage` |
| `is_in_editable_collection_by_user` | 是否在可编辑集合中（防权限提升） | `(!read_only && !hide_passwords) \|\| manage` |

**与端点权限的关系**：
- `put_cipher` / `post_cipher` → 使用 `is_write_accessible_to_user`
- `put_cipher_partial` → 使用 `is_accessible_to_user`（权限要求更低）

---

## 七、同步返回的相关关系

### 7.1 同步入口

**代码路径**: [sync](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L121-L204)

`GET /sync` 端点是客户端数据同步的核心入口。同步响应结构：

```json
{
    "profile": { ... },
    "folders": [ ... ],           // 用户的私人文件夹列表
    "collections": [ ... ],       // 用户可访问的组织集合
    "ciphers": [ ... ],           // 用户可见的所有密码条目
    "domains": { ... },
    "sends": [ ... ],
    "object": "sync"
}
```

**关键**：`folders` 和 `ciphers` 是两个独立的顶层数组，但密码条目中的 `folderId` 字段将两者关联起来。

### 7.2 CipherSyncData — 预加载个人状态

**代码路径**: [CipherSyncData](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L2100-L2214)

```rust
pub struct CipherSyncData {
    pub cipher_attachments: HashMap<CipherId, Vec<Attachment>>,
    pub cipher_folders: HashMap<CipherId, FolderId>,           // ← 个人状态
    pub cipher_favorites: HashSet<CipherId>,                   // ← 个人状态
    pub cipher_collections: HashMap<CipherId, Vec<CollectionId>>,
    pub cipher_archives: HashMap<CipherId, NaiveDateTime>,     // ← 个人状态
    pub members: HashMap<OrganizationId, Membership>,
    pub user_collections: HashMap<CollectionId, CollectionUser>,
    pub user_collections_groups: HashMap<CollectionId, CollectionGroup>,
    pub user_group_full_access_for_organizations: HashSet<OrganizationId>,
}
```

构建时按 `CipherSyncType` 区分加载策略：

| 同步类型 | cipher_folders | cipher_favorites | cipher_archives |
|---------|---------------|-----------------|----------------|
| `User` | ✓ 从 `FolderCipher::find_by_user` 加载 | ✓ 从 `Favorite::get_all_cipher_uuid_by_user` 加载 | ✓ 从 `Archive::find_by_user` 加载 |
| `Organization` | 空 HashMap | 空 HashSet | 空 HashMap |

组织同步不加载个人状态（[L2135-L2142](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/api/core/ciphers.rs#L2135-L2142)），注释说明：**如果设置了这些字段，会导致 web-vault 出现问题**。

### 7.3 to_json 中个人状态的注入

**代码路径**: [Cipher::to_json](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L370-L399)

```rust
if sync_type == CipherSyncType::User {
    json_object["folderId"] = json!(if let Some(cipher_sync_data) = cipher_sync_data {
        cipher_sync_data.cipher_folders.get(&self.uuid).cloned()  // 预加载 → O(1)
    } else {
        self.get_folder_uuid(user_uuid, conn).await               // 单独查询 → JOIN folders
    });

    json_object["favorite"] = json!(if let Some(cipher_sync_data) = cipher_sync_data {
        cipher_sync_data.cipher_favorites.contains(&self.uuid)    // 预加载 → O(1)
    } else {
        self.is_favorite(user_uuid, conn).await                   // 单独查询 → SELECT COUNT
    });

    json_object["archivedDate"] = json!(...);
    json_object["edit"] = json!(!read_only);
    json_object["viewPassword"] = json!(!hide_passwords);
    json_object["permissions"] = json!({
        "delete": !read_only,
        "restore": !read_only,
    });
}
```

**两条查询路径**：
- **批量同步**（`cipher_sync_data` 有值）：从预加载的 HashMap/HashSet 中 O(1) 查找
- **单条查询**（`cipher_sync_data` 为 None）：直接查询数据库

### 7.4 个人状态与条目权限的独立性

同步返回中，个人状态和条目权限是**完全独立**的字段：

```
Cipher JSON:
{
    "id": "cipher-uuid",
    "organizationId": "org-uuid",        // 是否为共享条目
    "folderId": "folder-uuid",           // 个人状态：文件夹归属（来自 folders_ciphers + folders.user_uuid）
    "favorite": true,                    // 个人状态：收藏（来自 favorites 表）
    "archivedDate": "2026-01-01T00:00",  // 个人状态：归档（来自 archives 表）
    "edit": false,                       // 权限状态：是否可编辑（来自 get_access_restrictions）
    "viewPassword": true,                // 权限状态：是否可查看密码
    "permissions": { "delete": false, "restore": false },
    ...条目内容字段...
}
```

**关键关系**：
- `edit: false`（只读）时，客户端仍可修改 `folderId` 和 `favorite`（通过 `/partial` 端点）
- `folderId` 为 `null` 表示密码不在任何文件夹中
- `favorite` 为 `false` 不代表用户"取消收藏"，仅表示当前不在收藏列表中
- 同一共享密码对不同用户返回不同的 `folderId`、`favorite`、`archivedDate` 值

### 7.5 密码删除时的个人状态清理

**代码路径**: [Cipher::delete](file:///d:/fz/0601/solo-dogfeeding/code/17-vaultwarden/src/db/models/cipher.rs#L476-L490)

```rust
pub async fn delete(&self, conn: &DbConn) -> EmptyResult {
    self.update_users_revision(conn).await;

    FolderCipher::delete_all_by_cipher(&self.uuid, conn).await?;    // 清理所有用户的文件夹关联
    CollectionCipher::delete_all_by_cipher(&self.uuid, conn).await?; // 清理集合关联
    Attachment::delete_all_by_cipher(&self.uuid, conn).await?;       // 清理附件
    Favorite::delete_all_by_cipher(&self.uuid, conn).await?;         // 清理所有用户的收藏

    diesel::delete(ciphers::table.filter(ciphers::uuid.eq(&self.uuid)))...
}
```

密码被删除时，**所有用户**的 FolderCipher 和 Favorite 记录都会被清理。

---

## 八、关键表关系与数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                        同步响应 (GET /sync)                      │
│                                                                  │
│  folders[]  ←── Folder::find_by_user() ── folders 表            │
│       │                                                          │
│       │  folderId 关联                                           │
│       ▼                                                          │
│  ciphers[]  ←── Cipher::find_by_user_visible()                  │
│       │         │                                                │
│       │         ├─ folderId    ← cipher_folders HashMap          │
│       │         │              ← FolderCipher + folders JOIN     │
│       │         │                                                │
│       │         ├─ favorite   ← cipher_favorites HashSet         │
│       │         │              ← favorites 表                    │
│       │         │                                                │
│       │         ├─ archivedDate ← cipher_archives HashMap        │
│       │         │              ← archives 表                     │
│       │         │                                                │
│       │         ├─ edit/viewPassword ← get_access_restrictions  │
│       │         │              ← membership + collections + groups│
│       │         │                                                │
│       │         └─ collectionIds ← cipher_collections HashMap    │
│       │                           ← ciphers_collections 表       │
│       ▼                                                          │
│  collections[] ← Collection::find_by_user_uuid()                │
└─────────────────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────────────────┐
│                    写入路径对比                                │
│                                                               │
│  PUT /ciphers/<id>         需要 is_write_accessible          │
│    └─ update_cipher_from_data()                              │
│         ├─ 修改 cipher 表字段                                 │
│         ├─ move_to_folder()  → folders_ciphers 表            │
│         ├─ set_favorite()    → favorites 表                   │
│         └─ set_archived_at() → archives 表                    │
│                                                               │
│  PUT /ciphers/<id>/partial  仅需 is_accessible               │
│    ├─ move_to_folder()      → folders_ciphers 表              │
│    └─ set_favorite()        → favorites 表                    │
│                                                               │
│  两者都校验: Folder::find_by_uuid_and_user(folder_id, user)  │
└──────────────────────────────────────────────────────────────┘
```

---

## 九、个人状态三表对比

| 特性 | folders_ciphers | favorites | archives |
|------|----------------|-----------|----------|
| 主键 | `(cipher_uuid, folder_uuid)` | `(user_uuid, cipher_uuid)` | `(user_uuid, cipher_uuid)` |
| user_uuid | 无（通过 JOIN folders 获取） | 直接存储 | 直接存储 |
| 数据量 | 每用户每密码最多1条 | 每用户每密码最多1条 | 每用户每密码最多1条 |
| 同步批量查询 | `FolderCipher::find_by_user` | `Favorite::get_all_cipher_uuid_by_user` | `Archive::find_by_user` |
| 返回类型 | `Vec<(CipherId, FolderId)>` → HashMap | `Vec<CipherId>` → HashSet | `Vec<(CipherId, NaiveDateTime)>` → HashMap |
| to_json 注入字段 | `folderId` | `favorite` | `archivedDate` |
| 可通过 partial 更新 | ✓ | ✓ | ✗（有专门的 archive/unarchive 端点） |
| 删除密码时清理 | `delete_all_by_cipher` | `delete_all_by_cipher` | （需检查 Archive 模型） |
| 触发修订版本更新 | ✓（在 move_to_folder 中） | ✓（在 set_favorite 中） | ✓（在 save/delete_by_cipher 中） |
