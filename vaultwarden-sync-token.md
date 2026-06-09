# Vaultwarden Sync Token 与增量拉取代码分析

## 1. 总体架构概览

Vaultwarden 的同步机制基于**时间戳游标**而非显式的 Sync Token 字符串。核心同步路径由三部分组成：

| 层次 | 机制 | 主要文件 |
|------|------|----------|
| 用户级游标 | `User.updated_at` 作为全局同步版本号 | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/user.rs) |
| 实体级时间戳 | 每个实体（Cipher/Folder/Send/...）各自维护 `updated_at` / `revision_date` | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs), [folder.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/folder.rs), [send.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/send.rs) |
| 实时推送 | WebSocket + Push Notification 通知变更 | [notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/notifications.rs), [push.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/push.rs) |

同步入口位于 [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L121-L204) 的 `sync` 函数。

---

## 2. 客户端同步游标（Sync Token）

### 2.1 游标载体：`User.updated_at`

Vaultwarden 没有独立的 "sync token" 字段，而是使用 `users.updated_at` 列作为用户的全局同步版本号。

**字段定义** ([user.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/user.rs#L29-L72))：

```rust
pub struct User {
    pub uuid: UserId,
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,  // ← 同步游标
    // ...
}
```

**游标获取 API** ([accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/accounts.rs#L1169-L1174))：

```rust
#[get("/accounts/revision-date")]
fn revision_date(headers: Headers) -> JsonResult {
    let revision_date = headers.user.updated_at.and_utc().timestamp_millis();
    Ok(Json(json!(revision_date)))
}
```

客户端调用此端点获取毫秒级时间戳，作为本地同步状态的游标。

### 2.2 游标的更新传播

当任何关联数据发生变化时，`User.updated_at` 会被级联更新。核心入口是 [update_users_revision](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs#L414-L440)：

**Cipher 变更时的传播**：

```rust
// cipher.rs L414-L440
pub async fn update_users_revision(&self, conn: &DbConn) -> Vec<UserId> {
    let mut user_uuids = Vec::new();
    match self.user_uuid {
        // 个人密码：直接更新所有者
        Some(ref user_uuid) => {
            User::update_uuid_revision(user_uuid, conn).await;
            user_uuids.push(user_uuid.clone());
        }
        // 组织密码：更新所有能访问该密文的组织成员
        None => {
            if let Some(ref org_uuid) = self.organization_uuid {
                let mut collection_users = Membership::find_by_cipher_and_org(...).await;
                if CONFIG.org_groups_enabled() {
                    let group_users = Membership::find_by_cipher_and_org_with_group(...).await;
                    collection_users.extend(group_users);
                }
                for member in collection_users {
                    User::update_uuid_revision(&member.user_uuid, conn).await;
                    user_uuids.push(member.user_uuid.clone());
                }
            }
        }
    }
    user_uuids
}
```

**底层更新实现** ([user.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/user.rs#L351-L386))：

```rust
pub async fn update_uuid_revision(uuid: &UserId, conn: &DbConn) {
    // 将 users.updated_at 设置为当前 UTC 时间
    Self::update_revision_impl(uuid, &Utc::now().naive_utc(), conn).await
}
```

**触发游标更新的场景**（各实体 save/delete 时都会调用）：

| 实体 | 触发位置 |
|------|----------|
| Cipher | [cipher.rs L442-L445](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs#L442-L445) `save()` → `update_users_revision()` |
| Folder | [folder.rs L75-L77](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/folder.rs#L75-L77) `save()` → `User::update_uuid_revision()` |
| Collection | [collection.rs L161-L162](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/collection.rs#L161-L162) `save()` → `update_users_revision()` |
| Send | 类似，`save()` 时更新 |
| Organization | 类似 |

---

## 3. 变更筛选机制

### 3.1 当前实现：全量拉取 + 客户端比对

**重要发现**：Vaultwarden 当前的 `/sync` 端点**不支持服务端增量筛选**。每次同步都会返回用户可见的全部数据。

同步入口 [ciphers.rs L121-L204](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L121-L204)：

```rust
#[get("/sync?<data..>")]
async fn sync(data: SyncData, headers: Headers, ..., conn: DbConn) -> JsonResult {
    let user_json = headers.user.to_json(&conn).await;

    // 关键：拉取用户可见的全部密文，没有时间戳过滤
    let mut ciphers = Cipher::find_by_user_visible(&headers.user.uuid, &conn).await;

    // 拉取用户的全部文件夹、集合、Send、策略等
    let collections = Collection::find_by_user_uuid(...).await;
    let folders_json = Folder::find_by_user(...).await;
    let sends_json = Send::find_by_user(...).await;
    let policies_json = OrgPolicy::find_confirmed_by_user(...).await;

    // 组装完整响应
    Ok(Json(json!({
        "profile": user_json,
        "folders": folders_json,
        "collections": collections_json,
        "policies": policies_json,
        "ciphers": ciphers_json,  // 全量
        "domains": domains_json,
        "sends": sends_json,
        "object": "sync"
    })))
}
```

客户端需要自行对比各实体的 `revisionDate` 字段来判断哪些需要更新。

### 3.2 乐观并发控制：`LastKnownRevisionDate`

虽然不支持服务端增量拉取，但在**写入**时有基于时间戳的冲突检测机制。

**请求字段定义** ([ciphers.rs L293-L299](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L293-L299))：

```rust
pub struct CipherData {
    // The revision datetime (in ISO 8601 format) of the client's local copy
    // of the cipher. This is used to prevent a client from updating a cipher
    // when it doesn't have the latest version, as that can result in data loss.
    last_known_revision_date: Option<String>,
    // ...
}
```

**冲突检测逻辑** ([ciphers.rs L420-L433](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L420-L433))：

```rust
// Check that the client isn't updating an existing cipher with stale data.
if ut != UpdateType::None
    && let Some(dt) = data.last_known_revision_date
{
    match NaiveDateTime::parse_from_str(&dt, "%+") {
        Err(err) => warn!("Error parsing LastKnownRevisionDate '{dt}': {err}"),
        // 服务端版本比客户端版本新 1 秒以上 → 拒绝更新
        Ok(dt) if cipher.updated_at.signed_duration_since(dt).num_seconds() > 1 => {
            err!("The client copy of this cipher is out of date. Resync the client and try again.")
        }
        Ok(_) => (),
    }
}
```

判断逻辑：如果 `server_updated_at - client_last_known > 1 秒`，则返回错误要求客户端重新同步。

### 3.3 创建 vs 更新的判断

在处理密文共享/更新时，通过 `last_known_revision_date` 是否存在来区分创建与更新操作，进而推送不同类型的通知：

[ciphers.rs L1064-L1072](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L1064-L1072)：

```rust
// When LastKnownRevisionDate is None, it is a new cipher, so send CipherCreate.
let ut = if let Some(ut) = override_ut {
    ut
} else if data.cipher.last_known_revision_date.is_some() {
    UpdateType::SyncCipherUpdate   // 已有密文更新
} else {
    UpdateType::SyncCipherCreate   // 新密文创建
};
```

### 3.4 软删除标记：`deleted_at`

已删除的密文不会立即从数据库移除，而是通过 `deleted_at` 标记：

[cipher.rs L340-L342](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs#L340-L342)：

```rust
"revisionDate": format_date(&self.updated_at),
"deletedDate": self.deleted_at.map_or(Value::Null, |d| Value::String(format_date(&d))),
```

定期清理任务通过 `find_deleted_before` 查找并彻底清除超过保留期的记录：

[cipher.rs L967-L973](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs#L967-L973)：

```rust
pub async fn find_deleted_before(dt: &NaiveDateTime, conn: &DbConn) -> Vec<Self> {
    ciphers::table.filter(ciphers::deleted_at.lt(dt)).load::<Self>(conn)
}
```

---

## 4. 实时变更通知（WebSocket / Push）

除了轮询 `/sync` 和 `/accounts/revision-date`，Vaultwarden 还通过 WebSocket 和 Push 通知客户端数据变化。

### 4.1 UpdateType 枚举

[notifications.rs L620-L654](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/notifications.rs#L620-L654)：

```rust
pub enum UpdateType {
    SyncCipherUpdate = 0,
    SyncCipherCreate = 1,
    SyncLoginDelete = 2,
    SyncFolderDelete = 3,
    SyncCiphers = 4,       // 批量变更（批量删除/归档等）
    SyncVault = 5,
    SyncOrgKeys = 6,
    SyncFolderCreate = 7,
    SyncFolderUpdate = 8,
    SyncSettings = 10,
    LogOut = 11,
    SyncSendCreate = 12,
    SyncSendUpdate = 13,
    SyncSendDelete = 14,
    AuthRequest = 15,
    AuthRequestResponse = 16,
    None = 100,
}
```

### 4.2 Cipher 更新推送

[notifications.rs L407-L454](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/notifications.rs#L407-L454)：

```rust
pub async fn send_cipher_update(
    &self, ut: UpdateType, cipher: &Cipher, user_ids: &[UserId],
    device: &Device, collection_uuids: Option<Vec<CollectionId>>, conn: &DbConn,
) {
    // 组织集合变更时：UserId 为 null，RevisionDate 为当前时间
    let (user_id, collection_uuids, revision_date) = if let Some(collection_uuids) = collection_uuids {
        (Value::Nil, Value::Array(...), serialize_date(Utc::now().naive_utc()))
    } else {
        (convert_option(cipher.user_uuid.as_deref()), Value::Nil, serialize_date(cipher.updated_at))
    };

    let data = create_update(
        vec![
            ("Id".into(), cipher.uuid.to_string().into()),
            ("UserId".into(), user_id),
            ("OrganizationId".into(), org_id),
            ("CollectionIds".into(), collection_uuids),
            ("RevisionDate".into(), revision_date),  // 关键：携带实体级时间戳
        ],
        ut,
        Some(device.uuid.clone()),  // 触发变更的设备，避免回推
    );

    // WebSocket 推送
    for uuid in user_ids {
        self.send_update(uuid, &data).await;
    }
    // Push 通知（仅单用户场景）
    if CONFIG.push_enabled() && user_ids.len() == 1 {
        push_cipher_update(ut, cipher, device, conn).await;
    }
}
```

### 4.3 消息格式（MessagePack）

[notifications.rs L562-L593](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/notifications.rs#L562-L593)：

```
[
    1,                       // MessageType.Invocation
    {},                      // Headers
    null,                    // InvocationId
    "ReceiveMessage",        // Target method
    [{
        "ContextId": acting_device_id,  // 触发变更的设备 ID（自己不用同步）
        "Type": ut as i32,              // UpdateType
        "Payload": {                    // 变更详情
            "Id": "cipher-uuid",
            "RevisionDate": timestamp,
            ...
        }
    }]
]
```

客户端收到推送后：
1. 检查 `ContextId` 是否为自身，若是则忽略
2. 从 `Payload.RevisionDate` 与本地版本比对
3. 调用 `/sync` 拉取最新数据（或直接根据 Payload 做局部更新）

---

## 5. 响应组装流程（全量同步）

### 5.1 CipherSyncData：批量预加载优化

为避免 N+1 查询问题，全量同步前先通过 `CipherSyncData` 一次性预加载所有关联数据。

**结构体定义** ([ciphers.rs L2096-L2110](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L2096-L2110))：

```rust
pub struct CipherSyncData {
    pub cipher_attachments: HashMap<CipherId, Vec<Attachment>>,   // 密文 → 附件列表
    pub cipher_folders: HashMap<CipherId, FolderId>,             // 密文 → 文件夹
    pub cipher_favorites: HashSet<CipherId>,                     // 收藏的密文 ID
    pub cipher_collections: HashMap<CipherId, Vec<CollectionId>>,// 密文 → 所属集合
    pub cipher_archives: HashMap<CipherId, NaiveDateTime>,       // 密文 → 归档时间
    pub members: HashMap<OrganizationId, Membership>,            // 组织 → 用户成员关系
    pub user_collections: HashMap<CollectionId, CollectionUser>, // 用户集合权限
    pub user_collections_groups: HashMap<CollectionId, CollectionGroup>, // 用户组集合权限
    pub user_group_full_access_for_organizations: HashSet<OrganizationId>,
}
```

**批量加载实现** ([ciphers.rs L2118-L2214](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L2118-L2214))，以几个典型查询为例：

```rust
pub async fn new(user_id: &UserId, sync_type: CipherSyncType, conn: &DbConn) -> Self {
    // 用户同步需要文件夹、收藏、归档信息；组织同步不需要
    match sync_type {
        CipherSyncType::User => {
            cipher_folders = FolderCipher::find_by_user(user_id, conn).await.into_iter().collect();
            cipher_favorites = Favorite::get_all_cipher_uuid_by_user(user_id, conn).await.into_iter().collect();
            cipher_archives = Archive::find_by_user(user_id, conn).await.into_iter().collect();
        }
        CipherSyncType::Organization => { /* 空 HashMap/HashSet */ }
    }

    // 批量查询附件
    let orgs = Membership::get_orgs_by_user(user_id, conn).await;
    let attachments = Attachment::find_all_by_user_and_orgs(user_id, &orgs, conn).await;

    // 批量查询集合关联
    let user_cipher_collections = Cipher::get_collections_with_cipher_by_user(user_id.clone(), conn).await;

    // 批量查询成员权限
    let members: HashMap<OrganizationId, Membership> = Membership::find_confirmed_by_user(user_id, conn)
        .await.into_iter().map(|m| (m.org_uuid.clone(), m)).collect();

    // ... 其他批量查询
}
```

### 5.2 单条 Cipher JSON 组装

[cipher.rs L145-L412](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs#L145-L412) 的 `to_json` 方法利用预加载数据快速构建响应：

```rust
pub async fn to_json(
    &self, host: &str, user_uuid: &UserId,
    cipher_sync_data: Option<&CipherSyncData>,  // 预加载数据
    sync_type: CipherSyncType, conn: &DbConn,
) -> Result<Value, Error> {
    // 1. 附件：优先使用预加载数据
    let attachments_json = if let Some(csd) = cipher_sync_data {
        csd.cipher_attachments.get(&self.uuid)  // O(1) HashMap 查找
    } else {
        Attachment::find_by_cipher(&self.uuid, conn).await  // 回退：单独查询
    };

    // 2. 集合 ID 列表
    let collection_ids = if let Some(csd) = cipher_sync_data {
        Cow::from(csd.cipher_collections.get(&self.uuid).unwrap_or(&vec![]))
    } else {
        Cow::from(self.get_admin_collections(user_uuid.clone(), conn).await)
    };

    // 3. 组装基础字段
    let mut json_object = json!({
        "object": "cipherDetails",
        "id": self.uuid,
        "type": self.atype,
        "creationDate": format_date(&self.created_at),
        "revisionDate": format_date(&self.updated_at),   // 实体级时间戳
        "deletedDate": self.deleted_at.map_or(Value::Null, |d| ...),
        "collectionIds": collection_ids,
        "attachments": attachments_json,
        // ... 其他字段
    });

    // 4. 用户级特有字段（仅用户同步）
    if sync_type == CipherSyncType::User {
        json_object["folderId"] = json!(cipher_sync_data.and_then(|csd| csd.cipher_folders.get(&self.uuid)));
        json_object["favorite"] = json!(cipher_sync_data.map_or(false, |csd| csd.cipher_favorites.contains(&self.uuid)));
        json_object["archivedDate"] = json!(cipher_sync_data.and_then(|csd| csd.cipher_archives.get(&self.uuid)));
        json_object["edit"] = json!(!read_only);
        json_object["viewPassword"] = json!(!hide_passwords);
    }

    Ok(json_object)
}
```

### 5.3 完整同步响应结构

[ciphers.rs L191-L203](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs#L191-L203)：

```json
{
    "profile": { /* User.to_json */ },
    "folders": [
        {
            "id": "...",
            "name": "...",
            "revisionDate": "2024-01-01T00:00:00.000Z",
            "object": "folder"
        }
    ],
    "collections": [
        {
            "id": "...",
            "organizationId": "...",
            "name": "...",
            "readOnly": false,
            "hidePasswords": false,
            "object": "collectionDetails"
        }
    ],
    "ciphers": [
        {
            "id": "...",
            "type": 1,
            "revisionDate": "2024-01-01T00:00:00.000Z",
            "deletedDate": null,
            "folderId": "...",
            "favorite": true,
            "collectionIds": ["..."],
            "object": "cipherDetails"
            // ...
        }
    ],
    "sends": [/* Send.to_json */],
    "policies": [/* OrgPolicy.to_json */],
    "domains": { /* equivalent domains */ },
    "userDecryption": { /* KDF params, encrypted user key */ },
    "object": "sync"
}
```

---

## 6. 完整同步时序

```
客户端                                  服务端 (Vaultwarden)
  │                                        │
  │──── GET /accounts/revision-date ──────▶│
  │                                        │ 查询 users.updated_at
  │◀─── 返回毫秒级时间戳 ──────────────────│
  │                                        │
  │  [本地比对：如时间戳无变化则跳过]       │
  │                                        │
  │──── GET /sync ────────────────────────▶│
  │                                        │
  │                                        │ 1. Cipher::find_by_user_visible()
  │                                        │    → 全量拉取用户可见密文
  │                                        │
  │                                        │ 2. CipherSyncData::new()
  │                                        │    → 批量预加载附件/文件夹/收藏/
  │                                        │      集合/权限等关联数据
  │                                        │
  │                                        │ 3. Folder::find_by_user()
  │                                        │    Collection::find_by_user_uuid()
  │                                        │    Send::find_by_user()
  │                                        │    OrgPolicy::find_confirmed_by_user()
  │                                        │
  │                                        │ 4. 逐项调用 to_json 组装
  │                                        │
  │◀─── 返回完整同步数据 ──────────────────│
  │                                        │
  │  [客户端按 revisionDate 逐条比对更新]   │
  │                                        │
  │────────────────────────────────────────│
  │         WebSocket 长连接                │
  │────────────────────────────────────────│
  │                                        │  [其他设备修改数据]
  │                                        │  → Cipher.save()
  │                                        │    → update_users_revision()
  │                                        │      → users.updated_at = NOW
  │                                        │  → send_cipher_update()
  │                                        │
  │◀── ReceiveMessage(UpdateType, Payload)─│
  │     {RevisionDate, Id, ...}            │
  │                                        │
  │  [根据推送决定是否重新 /sync]           │
```

---

## 7. 关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 全量同步入口 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs) | L121-L204 |
| 游标获取 API | [accounts.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/accounts.rs) | L1169-L1174 |
| User 游标更新 | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/user.rs) | L351-L386 |
| Cipher 游标传播 | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs) | L414-L445 |
| 写冲突检测（LastKnownRevisionDate） | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs) | L420-L433 |
| CipherSyncData 批量预加载 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/core/ciphers.rs) | L2096-L2214 |
| Cipher JSON 组装 | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/db/models/cipher.rs) | L145-L412 |
| WebSocket 推送（Cipher） | [notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/notifications.rs) | L407-L454 |
| UpdateType 枚举 | [notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/notifications.rs) | L620-L654 |
| Push 通知转发 | [push.rs](file:///d:/fz/0601/solo-dogfeeding/code/131-vaultwarden/src/api/push.rs) | L160-L189 |
