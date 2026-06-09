# Vaultwarden Sync Token 与增量拉取代码分析

## 1. 总体架构概览

Vaultwarden 的同步机制基于**时间戳游标**而非显式的 Sync Token 字符串。核心同步路径由三部分组成：

| 层次 | 机制 | 主要文件 |
|------|------|----------|
| 用户级游标 | `User.updated_at` 作为全局同步版本号 | `src/db/models/user.rs` |
| 实体级时间戳 | 每个实体（Cipher/Folder/Send/...）各自维护 `updated_at` / `revision_date` | `src/db/models/cipher.rs`, `src/db/models/folder.rs`, `src/db/models/send.rs` |
| 实时推送 | WebSocket + Push Notification 通知变更 | `src/api/notifications.rs`, `src/api/push.rs` |

同步入口位于 `src/api/core/ciphers.rs` 的 `sync` 函数（约 L121-L204）。

---

## 2. 客户端同步游标（Sync Token）

### 2.1 游标载体：`User.updated_at`

Vaultwarden 没有独立的 "sync token" 字段，而是使用 `users.updated_at` 列作为用户的全局同步版本号。

**字段定义**（`src/db/models/user.rs` User 结构体）：

```rust
pub struct User {
    pub uuid: UserId,
    pub created_at: NaiveDateTime,
    pub updated_at: NaiveDateTime,  // ← 同步游标
    // ...
}
```

**游标获取 API**（`src/api/core/accounts.rs` revision_date 函数）：

```rust
#[get("/accounts/revision-date")]
fn revision_date(headers: Headers) -> JsonResult {
    let revision_date = headers.user.updated_at.and_utc().timestamp_millis();
    Ok(Json(json!(revision_date)))
}
```

客户端调用此端点获取毫秒级时间戳，作为本地同步状态的游标。

### 2.2 游标的更新传播

当任何关联数据发生变化时，`User.updated_at` 会被级联更新。核心入口是 `Cipher::update_users_revision`（`src/db/models/cipher.rs`，约 L414-L440）：

**Cipher 变更时的传播**：

```rust
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

**底层更新实现**（`src/db/models/user.rs` update_uuid_revision 函数）：

```rust
pub async fn update_uuid_revision(uuid: &UserId, conn: &DbConn) {
    // 将 users.updated_at 设置为当前 UTC 时间
    Self::update_revision_impl(uuid, &Utc::now().naive_utc(), conn).await
}
```

**触发游标更新的场景**（各实体 save/delete 时都会调用）：

| 实体 | 触发位置 |
|------|----------|
| Cipher | `src/db/models/cipher.rs` `save()` → `update_users_revision()` |
| Folder | `src/db/models/folder.rs` `save()` → `User::update_uuid_revision()` |
| Collection | `src/db/models/collection.rs` `save()` → `update_users_revision()` |
| Send | 类似，`save()` 时更新 |
| Organization | 类似 |

---

## 3. 变更筛选机制

### 3.1 当前实现：全量拉取 + 客户端比对

**重要发现**：Vaultwarden 当前的 `/sync` 端点**不支持服务端增量筛选**。每次同步都会返回用户可见的全部数据。

同步入口（`src/api/core/ciphers.rs` sync 函数）：

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

### 3.2 响应级筛选：excludeDomains 与 SSH Key 隐藏

虽然没有基于时间戳的增量筛选，但 `/sync` 端点在响应组装前做了**两项条件筛选**。

#### 3.2.1 excludeDomains 参数对 domains 字段的影响

**请求参数定义**（`src/api/core/ciphers.rs` SyncData 结构体）：

```rust
#[derive(FromForm, Default)]
struct SyncData {
    #[field(name = "excludeDomains")]
    exclude_domains: bool, // Default: 'false'
}
```

**筛选逻辑**（sync 函数内部，组装 domains_json 处）：

```rust
let domains_json = if data.exclude_domains {
    Value::Null                              // excludeDomains=true → 返回 null
} else {
    api::core::get_eq_domains(&headers, true).into_inner()  // 否则返回等价域名数据
};
```

当 `excludeDomains=true` 时，响应中的 `domains` 字段直接为 `null`，跳过等价域名查询。

**域名数据来源**（`src/api/core/mod.rs` get_eq_domains 函数）：

```rust
fn get_eq_domains(headers: &Headers, no_excluded: bool) -> Json<Value> {
    let user = &headers.user;
    let equivalent_domains: Vec<Vec<String>> = from_str(&user.equivalent_domains).unwrap();
    let excluded_globals: Vec<i32> = from_str(&user.excluded_globals).unwrap();

    let mut globals: Vec<GlobalDomain> = from_str(GLOBAL_DOMAINS).unwrap();
    for global in &mut globals {
        global.excluded = excluded_globals.contains(&global.r#type);
    }

    // sync 调用时 no_excluded=true → 过滤掉用户已排除的全局域名组
    if no_excluded {
        globals.retain(|g| !g.excluded);
    }

    Json(json!({
        "equivalentDomains": equivalent_domains,
        "globalEquivalentDomains": globals,
        "object": "domains",
    }))
}
```

注意：同步接口调用 `get_eq_domains` 时第二个参数 `no_excluded=true`，意味着返回的 `globalEquivalentDomains` 还会额外过滤掉用户在设置中排除的全局域名组（与 `/settings/domains` 端点行为不同，后者保留全部并标记 excluded 字段）。

#### 3.2.2 低版本客户端隐藏 SSH Key

**客户端版本解析**（`src/auth.rs` ClientVersion 请求守卫）：

```rust
pub struct ClientVersion(pub semver::Version);

#[rocket::async_trait]
impl<'r> FromRequest<'r> for ClientVersion {
    async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error> {
        let headers = request.headers();

        // 从 HTTP Header "Bitwarden-Client-Version" 读取 semver 版本号
        let Some(version) = headers.get_one("Bitwarden-Client-Version") else {
            err_handler!("No Bitwarden-Client-Version header provided")
        };

        let Ok(version) = semver::Version::parse(version) else {
            err_handler!("Invalid Bitwarden-Client-Version header provided")
        };

        Outcome::Success(ClientVersion(version))
    }
}
```

sync 函数签名中 `client_version: Option<ClientVersion>`，意味着版本头是可选的。

**SSH Key 筛选逻辑**（sync 函数内部，约 L128-L137）：

```rust
// Filter out SSH keys if the client version is less than 2024.12.0
let show_ssh_keys = if let Some(client_version) = client_version {
    let ver_match = semver::VersionReq::parse(">=2024.12.0").unwrap();
    ver_match.matches(&client_version.0)
} else {
    false   // 未提供版本头 → 默认隐藏
};
if !show_ssh_keys {
    ciphers.retain(|c| c.atype != 5);  // atype=5 即 SshKey
}
```

完整判定表：

| 条件 | show_ssh_keys | 结果 |
|------|--------------|------|
| 未提供 `Bitwarden-Client-Version` Header | `false` | 过滤掉所有 `atype=5` 的密文 |
| 版本 `< 2024.12.0` | `false` | 过滤掉所有 `atype=5` 的密文 |
| 版本 `>= 2024.12.0` | `true` | 保留 SSH Key 类型密文 |

Cipher 类型定义（`src/db/models/cipher.rs`）：
- `1` = Login
- `2` = SecureNote
- `3` = Card
- `4` = Identity
- `5` = **SshKey**（SSH Key 是 v2024.12.0 引入的新类型）

### 3.3 乐观并发控制：`LastKnownRevisionDate`

虽然不支持服务端增量拉取，但在**写入**时有基于时间戳的冲突检测机制。

**请求字段定义**（`src/api/core/ciphers.rs` CipherData 结构体）：

```rust
pub struct CipherData {
    // The revision datetime (in ISO 8601 format) of the client's local copy
    // of the cipher. This is used to prevent a client from updating a cipher
    // when it doesn't have the latest version, as that can result in data loss.
    last_known_revision_date: Option<String>,
    // ...
}
```

**冲突检测逻辑**（`src/api/core/ciphers.rs` update_cipher_from_data 内部）：

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

### 3.4 创建 vs 更新的判断

在处理密文共享/更新时，通过 `last_known_revision_date` 是否存在来区分创建与更新操作，进而推送不同类型的通知（`src/api/core/ciphers.rs` share_cipher 内部）：

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

### 3.5 软删除标记：`deleted_at`

已删除的密文不会立即从数据库移除，而是通过 `deleted_at` 标记（`src/db/models/cipher.rs` to_json 方法）：

```rust
"revisionDate": format_date(&self.updated_at),
"deletedDate": self.deleted_at.map_or(Value::Null, |d| Value::String(format_date(&d))),
```

定期清理任务通过 `find_deleted_before` 查找并彻底清除超过保留期的记录（`src/db/models/cipher.rs`）：

```rust
pub async fn find_deleted_before(dt: &NaiveDateTime, conn: &DbConn) -> Vec<Self> {
    ciphers::table.filter(ciphers::deleted_at.lt(dt)).load::<Self>(conn)
}
```

---

## 4. 实时变更通知（WebSocket / Push）

除了轮询 `/sync` 和 `/accounts/revision-date`，Vaultwarden 还通过 WebSocket 和 Push 通知客户端数据变化。

### 4.1 UpdateType 枚举

（`src/api/notifications.rs` UpdateType 枚举）：

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

（`src/api/notifications.rs` WebSocketUsers::send_cipher_update 方法）：

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

（`src/api/notifications.rs` create_update 函数）：

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

**结构体定义**（`src/api/core/ciphers.rs` CipherSyncData 结构体）：

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

**批量加载实现**（`src/api/core/ciphers.rs` CipherSyncData::new），以几个典型查询为例：

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

`src/db/models/cipher.rs` 的 `to_json` 方法利用预加载数据快速构建响应：

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

（`src/api/core/ciphers.rs` sync 函数最终返回值）：

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
    "domains": { /* equivalent domains 或 null */ },
    "userDecryption": { /* KDF params, encrypted user key */ },
    "object": "sync"
}
```

---

## 6. 完整同步时序

```
客户端                                          服务端 (Vaultwarden)
  │                                                │
  │──── GET /accounts/revision-date ─────────────▶│
  │                                                │ 查询 users.updated_at
  │◀─── 返回毫秒级时间戳 ──────────────────────────│
  │                                                │
  │  [本地比对：如时间戳无变化则跳过]               │
  │                                                │
  │──── GET /sync?excludeDomains=false ───────────▶│
  │     Header: Bitwarden-Client-Version: 2025.1.0 │
  │                                                │
  │                                                │ 1. 解析 excludeDomains 参数（默认 false）
  │                                                │ 2. 解析客户端版本（≥2024.12.0 → 保留 SSH Key）
  │                                                │ 3. Cipher::find_by_user_visible()
  │                                                │    → 全量拉取用户可见密文
  │                                                │    → retain(|c| c.atype != 5) 可能过滤 SSH Key
  │                                                │
  │                                                │ 4. CipherSyncData::new()
  │                                                │    → 批量预加载附件/文件夹/收藏/
  │                                                │      集合/权限等关联数据
  │                                                │
  │                                                │ 5. Folder::find_by_user()
  │                                                │    Collection::find_by_user_uuid()
  │                                                │    Send::find_by_user()
  │                                                │    OrgPolicy::find_confirmed_by_user()
  │                                                │    get_eq_domains(no_excluded=true) 或 null
  │                                                │
  │                                                │ 6. 逐项调用 to_json 组装
  │                                                │
  │◀─── 返回完整同步数据 ──────────────────────────│
  │                                                │
  │  [客户端按 revisionDate 逐条比对更新]           │
  │                                                │
  │────────────────────────────────────────────────│
  │         WebSocket 长连接                        │
  │────────────────────────────────────────────────│
  │                                                │  [其他设备修改数据]
  │                                                │  → Cipher.save()
  │                                                │    → update_users_revision()
  │                                                │      → users.updated_at = NOW
  │                                                │  → send_cipher_update()
  │                                                │
  │◀── ReceiveMessage(UpdateType, Payload) ────────│
  │     {RevisionDate, Id, ContextId, ...}         │
  │                                                │
  │  [ContextId == 自身? 跳过 : 决定是否重新 /sync] │
```

---

## 7. 关键代码索引

| 功能 | 文件位置 | 区域 |
|------|----------|------|
| 全量同步入口 | `src/api/core/ciphers.rs` | sync 函数 |
| 游标获取 API | `src/api/core/accounts.rs` | revision_date 函数 |
| User 游标更新 | `src/db/models/user.rs` | update_uuid_revision / update_revision_impl |
| Cipher 游标传播 | `src/db/models/cipher.rs` | update_users_revision |
| excludeDomains 筛选 | `src/api/core/ciphers.rs` | sync 函数内 domains_json 分支 |
| 等价域名组装 | `src/api/core/mod.rs` | get_eq_domains 函数 |
| 客户端版本解析 | `src/auth.rs` | ClientVersion FromRequest impl |
| SSH Key 版本筛选 | `src/api/core/ciphers.rs` | sync 函数内 show_ssh_keys 分支 |
| 写冲突检测 LastKnownRevisionDate | `src/api/core/ciphers.rs` | update_cipher_from_data 内部 |
| CipherSyncData 批量预加载 | `src/api/core/ciphers.rs` | CipherSyncData struct + new 方法 |
| Cipher JSON 组装 | `src/db/models/cipher.rs` | to_json 方法 |
| WebSocket 推送 Cipher | `src/api/notifications.rs` | WebSocketUsers::send_cipher_update |
| UpdateType 枚举 | `src/api/notifications.rs` | UpdateType enum |
| Push 通知转发 | `src/api/push.rs` | push_cipher_update 等函数 |
