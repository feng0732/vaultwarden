# Send 分享和过期策略代码分析

## 一、整体架构概览

| 模块 | 文件路径 | 主要职责 |
|------|---------|---------|
| 数据模型 | [src/db/models/send.rs](src/db/models/send.rs) | Send 数据结构、密码验证、数据库操作 |
| API 接口 | [src/api/core/sends.rs](src/api/core/sends.rs) | Send 创建、访问、更新、删除接口 |
| 定时任务 | [src/main.rs](src/main.rs#L679-L684) | Send 过期清理定时任务调度 |
| 配置项 | [src/config.rs](src/config.rs#L543-L545) | Send 清理任务 cron 配置 |
| 存储抽象 | [src/storage.rs](src/storage.rs#L49-L76) | FS vs S3 存储后端判断与 operator 构建 |
| JWT 认证 | [src/auth.rs](src/auth.rs#L522-L528) | Send 下载链接 JWT token 生成与验证 |

---

## 二、数据模型核心字段

[src/db/models/send.rs#L23-L49](src/db/models/send.rs#L23-L49)

```rust
pub struct Send {
    pub uuid: SendId,                      // Send 唯一标识
    pub user_uuid: Option<UserId>,         // 创建用户
    pub organization_uuid: Option<OrganizationId>,

    pub password_hash: Option<Vec<u8>>,    // 密码哈希
    password_salt: Option<Vec<u8>>,        // 密码盐值
    password_iter: Option<i32>,            // 迭代次数 (100,000)

    pub max_access_count: Option<i32>,     // 最大访问次数
    pub access_count: i32,                 // 当前已访问次数

    pub creation_date: NaiveDateTime,
    pub expiration_date: Option<NaiveDateTime>,  // 过期时间 (可选)
    pub deletion_date: NaiveDateTime,            // 强制删除时间 (必填, ≤31天)

    pub disabled: bool,
}
```

---

## 三、访问条件检查流程

### 1. 三个公共端点

| 端点 | 功能 | 认证 | 查找方式 |
|------|------|------|---------|
| `POST /sends/access/<access_id>` | 文本/文件元数据获取 | 无 (匿名) | `access_id` → Base64URL 解码 → UUID → `find_by_uuid` |
| `POST /sends/<send_id>/access/file/<file_id>` | 文件下载链接获取 | 无 (匿名) | `send_id` 直接按 UUID 查找 |
| `GET /sends/<send_id>/<file_id>?<t>` | 本地存储模式实际下载 | JWT token | URL 参数 `t` 中的 JWT |

### 2. accessId 与 sendId 的关系

**accessId** 是 send UUID 的 Base64URL 编码：
[src/db/models/send.rs#L270-L281](src/db/models/send.rs#L270-L281)

```rust
pub async fn find_by_access_id(access_id: &str, conn: &DbConn) -> Option<Self> {
    let Ok(uuid_vec) = BASE64URL_NOPAD.decode(access_id.as_bytes()) else {
        return None;
    };
    let uuid = match Uuid::from_slice(&uuid_vec) {
        Ok(u) => SendId::from(u.to_string()),
        Err(_) => return None,
    };
    Self::find_by_uuid(&uuid, conn).await
}
```

`to_json` 中也展示了这个编码关系：
[src/db/models/send.rs#L150](src/db/models/send.rs#L150)
```rust
"accessId": BASE64URL_NOPAD.encode(Uuid::parse_str(&self.uuid).unwrap_or_default().as_bytes()),
```

**sendId** 就是原始 UUID 字符串。前端用 `accessId`（Base64URL 编码）调用 `post_access`，用 `sendId`（原始 UUID）调用 `post_access_file`。

### 3. 访问条件检查顺序

`post_access` 和 `post_access_file` 的检查逻辑**完全相同**，按以下顺序：

```
  ┌──────────────────────┐
  │ 1. Send 是否已存在?  │ → 404
  └──────────┬───────────┘
  ┌──────────▼───────────┐
  │ 2. 访问次数超限?     │ → 404
  │ access_count >= max  │
  └──────────┬───────────┘
  ┌──────────▼───────────┐
  │ 3. 是否已过期?       │ → 404
  │ now >= expiration    │
  └──────────┬───────────┘
  ┌──────────▼───────────┐
  │ 4. 是否到删除时间?   │ → 404
  │ now >= deletion_date │
  └──────────┬───────────┘
  ┌──────────▼───────────┐
  │ 5. 是否已禁用?       │ → 404
  └──────────┬───────────┘
  ┌──────────▼───────────┐
  │ 6. 密码验证 (如有)   │ → 401 / 错误
  └──────────┬───────────┘
             ✓ 通过
```

所有非密码失败统一返回 `"Send does not exist or is no longer available"` (404)，防止信息泄露。

---

## 四、文件分享的完整访问链路

### 链路总览：三步走

```
前端                         Vaultwarden                          存储后端
 │                              │                                    │
 │ ① POST /sends/access/<aid>  │                                    │
 ├─────────────────────────────►│                                    │
 │   6层检查通过                 │                                    │
 │   文本: +1计数, 返回内容      │                                    │
 │   文件: 不计数, 返回元数据    │                                    │
 │◄─────────────────────────────┤                                    │
 │   { file: { id: "fileId" } } │                                    │
 │                              │                                    │
 │ ② POST /sends/<sid>/access/file/<fid>                            │
 ├─────────────────────────────►│                                    │
 │   6层检查通过                 │                                    │
 │   +1计数, save, 生成下载URL   │                                    │
 │◄─────────────────────────────┤                                    │
 │   { url: "..." }             │                                    │
 │                              │                                    │
 │ ③ GET 下载URL                │                                    │
 ├─────────────────────────────►│ (FS模式)                           │
 │   或                         │   验证JWT, 读本地文件               │
 ├──────────────────────────────────────────────────────────────────►│ (S3模式)
 │                              │                          验证预签名 │
 │◄─────────────────────────────┤                                    │
 │        文件内容               │                                    │
```

### 第一步：post_access — 获取文件元数据

[src/api/core/sends.rs#L450-L507](src/api/core/sends.rs#L450-L507)

此端点同时服务文本和文件两种 Send。关键行为差异：

```rust
// Files are incremented during the download
if send.atype == SendType::Text as i32 {
    send.access_count += 1;
}
// 文件类型: 此处不计数!
```

返回的 `to_json_access` 中，文件 Send 包含 `file` 字段：
[src/db/models/send.rs#L173-L193](src/db/models/send.rs#L173-L193)

```rust
pub async fn to_json_access(&self, conn: &DbConn) -> Value {
    let mut data = serde_json::from_str::<LowerCase<Value>>(&self.data)
        .map(|d| d.data).unwrap_or_default();
    // ...
    json!({
        "id": self.uuid,
        "type": self.atype,
        "name": self.name,
        "text": if self.atype == SendType::Text as i32 { Some(&data) } else { None },
        "file": if self.atype == SendType::File as i32 { Some(&data) } else { None },
        //                                  ↑ data 中包含 { id: "fileId", size: ..., sizeName: ... }
        "expirationDate": self.expiration_date.as_ref().map(format_date),
        "creatorIdentifier": self.creator_identifier(conn).await,
        "object": "send-access",
    })
}
```

**前端从 `file.id` 提取 `fileId`，再调用第二步**。

### 第二步：post_access_file — 获取下载链接

[src/api/core/sends.rs#L509-L568](src/api/core/sends.rs#L509-L568)

**核心执行顺序（极易出错）：**

```rust
async fn post_access_file(send_id, file_id, data, host, conn, nt) -> JsonResult {
    // 1. 查找 Send (用 send_id，不验证 file_id 归属!)
    let Some(mut send) = Send::find_by_uuid(&send_id, &conn).await else { ... };

    // 2. 6层访问检查 ...

    // 3. 访问次数 +1
    send.access_count += 1;

    // 4. 持久化到数据库
    send.save(&conn).await?;           // ← 此时计数已落盘

    // 5. 推送同步通知
    nt.send_send_update(...).await;

    // 6. 生成下载 URL
    Ok(Json(json!({
        "object": "send-fileDownload",
        "id": file_id,
        "url": download_url(&host, &send_id, &file_id).await?,  // ← 如果这里失败?
    })))
}
```

**关键发现：访问次数先落盘，下载链接后生成**

- 步骤 4 `send.save()` 成功后，`access_count + 1` 已经写入数据库
- 步骤 6 `download_url()` 如果失败（如 S3 连接异常、operator 构建失败），**次数已被消耗但未返回下载链接**
- Rust 的 `?` 传播错误时不会回滚 `save`，因为 `save` 和 `download_url` 不在同一事务中

> ⚠️ **结论：下载链接生成失败时，访问次数已被消耗且无法恢复。**

### 第三步：实际下载（仅 FS 模式经过后端）

[src/api/core/sends.rs#L583-L591](src/api/core/sends.rs#L583-L591)

```rust
#[get("/sends/<send_id>/<file_id>?<t>")]
async fn download_send(send_id: SendId, file_id: SendFileId, t: &str) -> Option<NamedFile> {
    if let Ok(claims) = crate::auth::decode_send(t)
        && claims.sub == format!("{send_id}/{file_id}")
    {
        return NamedFile::open(Path::new(&CONFIG.sends_folder()).join(send_id).join(file_id)).await.ok();
    }
    None
}
```

此端点**不检查** Send 的任何业务状态（访问次数、过期、禁用、密码），仅验证 JWT token。

---

## 五、fileId 归属校验分析

**`post_access_file` 不校验 `file_id` 是否属于该 Send。**

对比上传时的校验：

| 端点 | file_id 校验 |
|------|-------------|
| `post_send_file_v2_data`（上传） | ✅ 从 `send.data` 解析出 `SendFileData.id`，与请求中的 `file_id` 比对 |
| `post_access_file`（下载） | ❌ **无任何归属校验** |
| `download_send`（本地下载） | ❌ 仅验证 JWT 中的 `sub == {send_id}/{file_id}`，不校验归属 |

`post_access_file` 只用 `send_id` 查找 Send，`file_id` 直接透传给 `download_url` 和返回给前端。攻击者若知道一个合法 `send_id`，可以传入任意 `file_id`，最终获取的下载 URL 将指向 `{send_id}/{任意file_id}` 路径。但实际风险有限：

1. **FS 模式**：JWT 中绑定 `{send_id}/{file_id}`，下载时按此路径读文件，文件不存在则返回 `None`
2. **S3 模式**：预签名 URL 同样指向 `{send_id}/{file_id}`，S3 返回 404 如果对象不存在
3. 真正的文件列表存储在 `send.data` JSON 中，前端只展示 `data.id` 对应的文件

---

## 六、本地存储与 S3 预签名下载分支详解

### 存储后端判断

[src/storage.rs#L49-L51](src/storage.rs#L49-L51)
[src/storage.rs#L53-L76](src/storage.rs#L53-L76)

```rust
pub(crate) fn is_fs_operator(operator: &opendal::Operator) -> bool {
    operator.info().scheme() == opendal::services::FS_SCHEME
}

pub(crate) fn operator_for_path(path: &str) -> Result<opendal::Operator, crate::Error> {
    // 缓存机制: 已构建的 operator 按 path 缓存
    let operator = if path.starts_with("s3://") {
        // S3 路径 → 构建 S3 operator (需要 s3 feature)
        s3::operator_for_path(path)?
    } else {
        // 其他路径 → 构建本地文件系统 operator
        let builder = opendal::services::Fs::default().root(path);
        opendal::Operator::new(builder)?.finish()
    };
}
```

**判断依据**：`sends_folder()` 的返回值是否以 `s3://` 开头。

[src/config.rs#L513](src/config.rs#L513)
```rust
sends_folder: String, false, auto, |c| storage::join_path(&c.data_folder, "sends");
```

- 如果 `DATA_FOLDER` 是本地路径（如 `data`），`sends_folder` = `data/sends` → **FS 模式**
- 如果 `DATA_FOLDER` 以 `s3://` 开头，`sends_folder` = `s3://bucket/.../sends` → **S3 模式**

### 分支一：本地文件系统模式 (FS)

| 特性 | 说明 |
|------|------|
| URL 类型 | `https://{host}/api/sends/{send_id}/{file_id}?t={jwt}` |
| 认证方式 | JWT Token（2 分钟有效） |
| 下载流程 | 请求经过 Vaultwarden → 验证 JWT → 读取本地文件返回 |
| 文件路径 | `{sends_folder}/{send_id}/{file_id}` |
| 有效期内重复下载 | ✅ 可以（JWT 有效期内） |

JWT claims：
[src/auth.rs#L522-L528](src/auth.rs#L522-L528)
```rust
BasicJwtClaims {
    nbf: time_now.timestamp(),
    exp: (time_now + TimeDelta::try_minutes(2).unwrap()).timestamp(),  // 2分钟
    iss: JWT_SEND_ISSUER.to_string(),
    sub: format!("{send_id}/{file_id}"),
}
```

下载验证：
[src/api/core/sends.rs#L584-L591](src/api/core/sends.rs#L584-L591)
- `decode_send(t)` 验证签名 + 过期时间
- `claims.sub == format!("{send_id}/{file_id}")` 防止 token 被挪用
- 不检查 Send 业务状态

### 分支二：S3 预签名模式

| 特性 | 说明 |
|------|------|
| URL 类型 | S3 预签名 URL（直连 S3 服务） |
| 认证方式 | S3 签名（5 分钟有效） |
| 下载流程 | 请求不经过 Vaultwarden，直连 S3 |
| 对象路径 | `{sends_folder}/{send_id}/{file_id}`（S3 key） |
| 有效期内重复下载 | ✅ 可以（预签名有效期内） |

```rust
operator.presign_read(
    &format!("{send_id}/{file_id}"),
    Duration::from_mins(5)  // 5分钟有效期
).await?.uri().to_string()
```

> 注意：`presign_read` 的路径是相对 operator root 的。operator root 已经是 `sends_folder`，所以最终 S3 key 为 `{sends_folder前缀}/{send_id}/{file_id}`。

### 两条分支对比

| 对比项 | FS 本地模式 | S3 预签名模式 |
|--------|------------|--------------|
| 下载链接域名 | Vaultwarden 自身 | S3 服务域名 |
| 请求经过后端 | ✅ | ❌ |
| 认证方式 | JWT (2min) | S3 签名 (5min) |
| 下载时检查 Send 状态 | ❌ | ❌ |
| 下载流量 | 走 Vaultwarden | 走 S3 |
| 后端故障影响下载 | ✅ 影响 | ❌ 不影响 |
| 计数时机 | 获取链接时 +1 | 获取链接时 +1 |
| 链接生成失败消耗次数 | ✅ 是 | ✅ 是 |

---

## 七、密码保护机制

### 密码设置
[src/db/models/send.rs#L99-L113](src/db/models/send.rs#L99-L113)

- PBKDF2，100,000 次迭代
- 64 字节随机盐
- 传 `None` 时清除密码

### 密码验证
[src/db/models/send.rs#L115-L122](src/db/models/send.rs#L115-L122)

- 三元组 (`hash`, `salt`, `iter`) 缺一即返回 `false`
- `post_access` 中密码错误记录 IP
- `post_access_file` 中密码错误**不记录 IP**（两者日志策略不一致）

---

## 八、访问次数控制

### 计数递增时机

| Send 类型 | 计数端点 | 计数时机 | 代码位置 |
|----------|---------|---------|---------|
| 文本 (Text) | `post_access` | 访问验证通过后 | [sends.rs#L491-L493](src/api/core/sends.rs#L491-L493) |
| 文件 (File) | `post_access_file` | 获取下载链接时 | [sends.rs#L550](src/api/core/sends.rs#L550) |

**文本类型**：`post_access` 中直接计数并返回内容，一步完成。

**文件类型**：`post_access` 中**不计数**（代码注释 `Files are incremented during the download`），计数推迟到 `post_access_file`。

### 次数消耗与链接生成的顺序问题

`post_access_file` 中的精确执行顺序：

```
1. access_count += 1        ← 内存中计数
2. send.save(&conn)         ← 持久化到数据库（计数已落盘）
3. nt.send_send_update()    ← 推送同步通知
4. download_url().await?    ← 生成下载链接（可能失败!）
```

**如果步骤 4 失败**：
- 步骤 2 已提交，数据库中 `access_count` 已 +1
- Rust `?` 直接返回错误，不回滚
- **次数被消耗，但前端未获得下载链接**
- 用户重试将再次触发 `post_access_file`，再次 +1

> 这是一个**先记账后发货**的设计，发货失败不退账。

---

## 九、过期删除机制

### 两个时间维度

| 字段 | 性质 | 作用 | 限制 |
|------|------|------|------|
| `expiration_date` | 可选 | 到达后不可访问，数据仍在 | 无强制限制 |
| `deletion_date` | **必填** | 到达后从数据库彻底删除 | **≤ 31 天** |

### 删除日期强制限制
[src/api/core/sends.rs#L145-L149](src/api/core/sends.rs#L145-L149)

```rust
if data.deletion_date > Utc::now() + TimeDelta::try_days(31).unwrap() {
    err!("You cannot have a Send with a deletion date that far into the future. ...");
}
```

### 定时清理链路

```
cron "0 5 * * * *" (每小时第5分钟)
  → main.rs: Job 调度
    → sends.rs: purge_sends(pool)
      → send.rs: Send::purge(conn)
        → Send::find_by_past_deletion_date(conn)
            WHERE deletion_date < now()
        → 遍历: send.delete(conn)
            ├─ File类型: operator.delete_with(&self.uuid).recursive(true)  // 删除 {sends_folder}/{send_id}/ 下所有文件
            └─ diesel::delete(sends::table.filter(uuid.eq(&self.uuid)))    // 删除数据库记录
```

[src/db/models/send.rs#L246-L250](src/db/models/send.rs#L246-L250)

清理**只看 `deletion_date`**，不看 `expiration_date`。过期的 Send 如果 `deletion_date` 还未到，只是访问返回 404，数据仍然存在。

---

## 十、策略总结

### 访问条件逻辑真值表

| 存在 | 未超限 | 未过期 | 未到删除时间 | 未禁用 | 密码 | 结果 |
|-----|--------|--------|-------------|--------|------|------|
| ❌ | - | - | - | - | - | 404 |
| ✅ | ❌ | - | - | - | - | 404 |
| ✅ | ✅ | ❌ | - | - | - | 404 |
| ✅ | ✅ | ✅ | ❌ | - | - | 404 |
| ✅ | ✅ | ✅ | ✅ | ❌ | - | 404 |
| ✅ | ✅ | ✅ | ✅ | ✅ | 无/正确 | ✅ |
| ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ 密码错误 |

### 关键设计决策与风险点

1. **双重时间控制**：`expiration_date` 控制访问，`deletion_date` 控制清理，互不干扰
2. **最大 31 天**：deletion_date 不可超过创建时间 +31 天
3. **统一 404 错误**：防止信息泄露
4. **先记账后发货**：访问次数在生成下载链接前落盘，链接生成失败次数不退回
5. **fileId 无归属校验**：`post_access_file` 不验证 file_id 是否属于该 Send
6. **下载端点无业务检查**：`download_send` 仅验证 JWT，不检查过期/禁用/次数
7. **文本和文件计数分离**：文本在 `post_access` 计数，文件在 `post_access_file` 计数
8. **密码错误日志不一致**：`post_access` 记录 IP，`post_access_file` 不记录
