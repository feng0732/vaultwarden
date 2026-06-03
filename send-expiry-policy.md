# Send 分享和过期策略代码分析

## 一、整体架构概览

Send 是 Vaultwarden 的文件/文本临时分享功能，其访问控制和过期策略涉及以下核心文件：

| 模块 | 文件路径 | 主要职责 |
|------|---------|---------|
| 数据模型 | [src/db/models/send.rs](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/db/models/send.rs) | Send 数据结构、密码验证、数据库操作 |
| API 接口 | [src/api/core/sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs) | Send 创建、访问、更新、删除接口 |
| 定时任务 | [src/main.rs](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/main.rs#L679-L684) | Send 过期清理定时任务调度 |
| 配置项 | [src/config.rs](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/config.rs#L543-L545) | Send 清理任务 cron 配置 |
| 存储抽象 | [src/storage.rs](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/storage.rs#L49-L51) | 本地文件系统 vs S3 等存储后端判断 |
| JWT 认证 | [src/auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/auth.rs#L522-L528) | Send 下载链接 JWT token 生成与验证 |

---

## 二、数据模型核心字段

### Send 结构体关键字段
[src/db/models/send.rs#L23-L49](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/db/models/send.rs#L23-L49)

```rust
pub struct Send {
    pub uuid: SendId,                      // Send 唯一标识
    pub user_uuid: Option<UserId>,         // 创建用户
    pub organization_uuid: Option<OrganizationId>,

    // 密码保护相关
    pub password_hash: Option<Vec<u8>>,    // 密码哈希
    password_salt: Option<Vec<u8>>,        // 密码盐值
    password_iter: Option<i32>,            // 迭代次数 (100,000)

    // 访问次数控制
    pub max_access_count: Option<i32>,     // 最大访问次数
    pub access_count: i32,                 // 当前已访问次数

    // 时间控制
    pub creation_date: NaiveDateTime,      // 创建时间
    pub expiration_date: Option<NaiveDateTime>,  // 过期时间 (可选)
    pub deletion_date: NaiveDateTime,      // 强制删除时间 (必填, 最长31天)

    pub disabled: bool,                    // 是否禁用
}
```

---

## 三、访问条件检查流程

### 1. 公共访问入口
访问 Send 有两个主要入口：

| 端点 | 功能 | 位置 |
|------|------|------|
| `POST /sends/access/<access_id>` | 文本 Send 访问 | [sends.rs#L450-L507](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L450-L507) |
| `POST /sends/<send_id>/access/file/<file_id>` | 文件 Send 访问（获取下载链接） | [sends.rs#L509-L568](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L509-L568) |

> **重要区别**：文件访问接口只是获取下载链接，**不直接返回文件内容**。实际下载需要通过第二个请求完成。

### 2. 访问条件检查顺序
两个入口的检查逻辑**完全相同**，按以下顺序依次检查：

```
                              ┌───────────────────────┐
                              │  1. Send 是否已存在?   │
                              │  find_by_access_id    │
                              └───────────┬───────────┘
                                          │ 不存在 → 404
                              ┌───────────▼───────────┐
                              │  2. 访问次数超限?     │
                              │  access_count >= max  │
                              └───────────┬───────────┘
                                          │ 超限 → 404
                              ┌───────────▼───────────┐
                              │  3. 是否已过期?       │
                              │  now >= expiration    │
                              └───────────┬───────────┘
                                          │ 过期 → 404
                              ┌───────────▼───────────┐
                              │  4. 是否已到删除时间? │
                              │  now >= deletion_date │
                              └───────────┬───────────┘
                                          │ 到点 → 404
                              ┌───────────▼───────────┐
                              │  5. 是否已禁用?       │
                              │  disabled == true     │
                              └───────────┬───────────┘
                                          │ 禁用 → 404
                              ┌───────────▼───────────┐
                              │  6. 密码验证 (如设置) │
                              │  check_password()     │
                              └───────────┬───────────┘
                                          │ 错误 → 401/错误
                              ┌───────────▼───────────┐
                              │  ✓ 访问成功，计数+1   │
                              └───────────────────────┘
```

**关键代码示例**：
```rust
// 检查 1-5: 所有失败都返回相同的 404 错误，避免信息泄露
const SEND_INACCESSIBLE_MSG: &str = "Send does not exist or is no longer available";

// 检查 6: 密码验证
if send.password_hash.is_some() {
    match data.into_inner().password {
        Some(ref p) if send.check_password(p) => { /* 通过 */ }
        Some(_) => err!("Invalid password", format!("IP: {}.", ip.ip)),
        None => err_code!("Password not provided", ..., 401),
    }
}
```

### 3. 安全设计要点
- **统一错误信息**：条件 1-5 全部返回相同的 404 错误信息，防止攻击者通过错误差异推断 Send 存在性
- **IP 日志**：密码验证失败时记录访问者 IP
- **分级错误码**：密码未提供返回 401，密码错误返回普通错误

---

## 四、密码保护机制

### 1. 密码设置
[src/db/models/send.rs#L99-L113](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/db/models/send.rs#L99-L113)

```rust
pub fn set_password(&mut self, password: Option<&str>) {
    const PASSWORD_ITER: i32 = 100_000;  // PBKDF2 迭代次数

    if let Some(password) = password {
        self.password_iter = Some(PASSWORD_ITER);
        let salt = crate::crypto::get_random_bytes::<64>().to_vec();  // 64字节随机盐
        let hash = crate::crypto::hash_password(password.as_bytes(), &salt, PASSWORD_ITER as u32);
        self.password_salt = Some(salt);
        self.password_hash = Some(hash);
    } else {
        // 清除密码
        self.password_iter = None;
        self.password_salt = None;
        self.password_hash = None;
    }
}
```

### 2. 密码验证
[src/db/models/send.rs#L115-L122](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/db/models/send.rs#L115-L122)

```rust
pub fn check_password(&self, password: &str) -> bool {
    match (&self.password_hash, &self.password_salt, self.password_iter) {
        (Some(hash), Some(salt), Some(iter)) => {
            crate::crypto::verify_password_hash(password.as_bytes(), salt, hash, iter.cast_unsigned())
        }
        _ => false,
    }
}
```

---

## 五、访问次数控制

### 1. 计数递增时机（**已纠正**）

| Send 类型 | 计数时机 | 代码位置 | 说明 |
|----------|---------|---------|------|
| **文本类型 (Text)** | 访问验证通过后立即 +1 | [sends.rs#L491-L493](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L491-L493) | 内容直接返回，访问即消耗一次 |
| **文件类型 (File)** | **获取下载链接时 +1** | [sends.rs#L550](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L550) | **关键点**：在 `post_access_file` 中计数+1，而非实际下载时 |

```rust
// 文本 Send: 访问即计数 (post_access)
if send.atype == SendType::Text as i32 {
    send.access_count += 1;
}

// 文件 Send: 获取下载链接时计数 (post_access_file)
// 注意：无论用户是否真正点击下载，只要成功获取下载链接就计数+1
send.access_count += 1;
```

### 2. 关键纠正：文件下载的"计数时机"与"实际下载"分离

**之前的误解**：文件下载时才计数

**实际逻辑**：
- 第一步：调用 `post_access_file` 获取下载链接 → **此时已计数 +1**
- 第二步：浏览器通过下载链接请求实际文件 → **不再检查访问次数，也不再计数**

> ⚠️ **重要行为**：用户点击"获取下载链接"按钮即消耗一次访问次数，即使他取消下载或下载失败。这是设计决策，避免需要在下载完成时回调增加复杂度。

---

## 六、文件下载完整链路分析

### 1. 整体流程图

```
用户前端                          Vaultwarden 后端                          存储后端
   │                                  │                                      │
   │ POST /sends/<id>/access/file/    │                                      │
   ├─────────────────────────────────►│                                      │
   │                                  │ 1. 6层访问检查 (存在/次数/过期等)     │
   │                                  │ 2. access_count += 1                 │
   │                                  │ 3. 调用 download_url()               │
   │                                  │                                      │
   │          返回下载 URL             │◄─────────────────────────────────────┤
   │◄─────────────────────────────────┤                                      │
   │                                  │                                      │
   │                                  │ ▼ 分支判断                           │
   │                                  │   is_fs_operator?                    │
   │                                  │   ├─ 是 → 本地 JWT 模式              │
   │                                  │   └─ 否 → S3 预签名模式              │
   │                                  │                                      │
   │     浏览器访问下载 URL            │                                      │
   ├─────────────────────────────────►│ (仅本地模式经过这里)                   │
   │                                  │ 验证 JWT token                       │
   │                                  │ 有效 → 读取本地文件并返回             │
   │          文件内容                 │                                      │
   │◄─────────────────────────────────┤                                      │
   │                                  │                                      │
   │                                  │                                      │
   │     (S3 模式: 直接跳 S3)          │                                      │
   ├────────────────────────────────────────────────────────────────────────►│
   │                                                                         │ 验证预签名 URL
   │                                                                         │ 有效 → 返回文件
   │          文件内容                                                      │
   │◄────────────────────────────────────────────────────────────────────────┤
```

### 2. 下载链接生成核心逻辑
[src/api/core/sends.rs#L570-L581](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L570-L581)

```rust
async fn download_url(host: &Host, send_id: &SendId, file_id: &SendFileId) -> Result<String, crate::Error> {
    let operator = CONFIG.opendal_operator_for_path_type(&PathType::Sends)?;

    if crate::storage::is_fs_operator(&operator) {
        // 分支1: 本地文件系统模式 → 生成带 JWT token 的内部 URL
        let token_claims = crate::auth::generate_send_claims(send_id, file_id);
        let token = crate::auth::encode_jwt(&token_claims);
        Ok(format!("{}/api/sends/{send_id}/{file_id}?t={token}", host.host))
    } else {
        // 分支2: 远程存储 (S3等) → 生成预签名 URL
        Ok(operator.presign_read(&format!("{send_id}/{file_id}"), Duration::from_mins(5)).await?.uri().to_string())
    }
}
```

---

## 七、两条下载分支详细对比

### 存储后端判断
[src/storage.rs#L49-L51](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/storage.rs#L49-L51)

```rust
pub(crate) fn is_fs_operator(operator: &opendal::Operator) -> bool {
    operator.info().scheme() == opendal::services::FS_SCHEME
}
```

判断依据：OpenDAL operator 的 scheme 是否为 `fs`（本地文件系统）。

---

### 分支一：本地文件系统模式 (FS)

| 特性 | 说明 |
|------|------|
| **URL 类型** | 内部 API URL，指向 Vaultwarden 自身 |
| **认证方式** | JWT Token |
| **Token 有效期** | **2 分钟** |
| **Token 内容** | `sub = {send_id}/{file_id}` |
| **下载流程** | 必须经过 Vaultwarden 后端 |
| **适用场景** | 默认本地存储、挂载存储 |

#### JWT Token 生成
[src/auth.rs#L522-L528](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/auth.rs#L522-L528)
```rust
pub fn generate_send_claims(send_id: &SendId, file_id: &SendFileId) -> BasicJwtClaims {
    let time_now = Utc::now();
    BasicJwtClaims {
        nbf: time_now.timestamp(),
        exp: (time_now + TimeDelta::try_minutes(2).unwrap()).timestamp(),  // 2分钟有效期
        iss: JWT_SEND_ISSUER.to_string(),
        sub: format!("{send_id}/{file_id}"),  // 绑定具体的 send 和 file
    }
}
```

#### 本地下载验证
[src/api/core/sends.rs#L583-L591](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L583-L591)
```rust
#[get("/sends/<send_id>/<file_id>?<t>")]
async fn download_send(send_id: SendId, file_id: SendFileId, t: &str) -> Option<NamedFile> {
    // 验证 JWT token: 1. 签名有效 2. 未过期 3. sub 匹配
    if let Ok(claims) = crate::auth::decode_send(t)
        && claims.sub == format!("{send_id}/{file_id}")
    {
        // Token 有效，直接读取本地文件返回
        return NamedFile::open(Path::new(&CONFIG.sends_folder()).join(send_id).join(file_id)).await.ok();
    }
    None
}
```

> **注意**：`download_send` **不检查** Send 的访问次数、过期时间、删除时间、禁用状态、密码！
>
> 这些检查只在获取下载链接时（`post_access_file`）做一次。JWT 有效期内可以重复下载。

---

### 分支二：远程存储模式 (S3 等)

| 特性 | 说明 |
|------|------|
| **URL 类型** | S3 预签名 URL，直接指向存储服务 |
| **认证方式** | S3 预签名机制 |
| **URL 有效期** | **5 分钟** |
| **下载流程** | 直接访问 S3，不经过 Vaultwarden |
| **适用场景** | S3、OSS 等对象存储 |

#### 预签名 URL 生成
```rust
// OpenDAL 自动调用对应存储后端的预签名 API
operator.presign_read(
    &format!("{send_id}/{file_id}"), 
    Duration::from_mins(5)  // 5分钟有效期
).await?.uri().to_string()
```

---

### 两条分支对比表

| 对比项 | 本地文件系统模式 | S3 预签名模式 |
|--------|-----------------|---------------|
| **下载链接域名** | Vaultwarden 自身域名 | S3 服务域名 |
| **请求是否经过后端** | ✅ 经过 | ❌ 不经过 (直连 S3) |
| **认证方式** | JWT Token | S3 预签名 |
| **有效期** | 2 分钟 | 5 分钟 |
| **有效期内重复下载** | ✅ 可以 | ✅ 可以 |
| **下载时再次检查 Send 状态** | ❌ 不检查 | ❌ 不检查 |
| **下载流量** | 走 Vaultwarden 带宽 | 走 S3 带宽 |
| **计数时机** | 获取链接时 +1 | 获取链接时 +1 |

---

## 八、文件分享时序图详解

```
用户浏览器                     post_access_file                download_send / S3
     │                              │                              │
     │ 1. 点击"下载"按钮             │                              │
     ├─────────────────────────────►│                              │
     │                              │                              │
     │                              │ 2. 🔍 6层访问检查             │
     │                              │    ├─ Send 存在?              │
     │                              │    ├─ 访问次数 < max?          │
     │                              │    ├─ 未过期?                 │
     │                              │    ├─ 未到删除时间?           │
     │                              │    ├─ 未禁用?                 │
     │                              │    └─ 密码正确? (如设置)      │
     │                              │                              │
     │                              │ 3. ⬆️ access_count += 1      │
     │                              │    (关键点：这里就计数了!)    │
     │                              │                              │
     │                              │ 4. 🎫 生成下载凭证           │
     │                              │    ├─ FS: JWT Token (2min)    │
     │                              │    └─ S3: 预签名 URL (5min)   │
     │                              │                              │
     │ 5. 返回下载 URL              │                              │
     │◄─────────────────────────────┤                              │
     │                              │                              │
     │ 6. 浏览器自动/手动访问下载URL │                              │
     ├────────────────────────────────────────────────────────────►│
     │                                                              │
     │                                                              │ 7. 验证下载凭证
     │                                                              │    ├─ FS: 验证 JWT
     │                                                              │    └─ S3: 验证预签名
     │                                                              │
     │ 8. 返回文件内容                                              │
     │◄─────────────────────────────────────────────────────────────┤
     │                                                              │
     │                                                              │
     │ ⚠️  此时用户可以:                                            │
     │    - 正常下载完成 ✓                                          │
     │    - 取消下载 ❌ (但次数已消耗!)                             │
     │    - 有效期内重复下载多次                                    │
```

---

## 九、过期删除机制

### 1. 两个时间维度

| 字段 | 性质 | 作用 | 限制 |
|------|------|------|------|
| `expiration_date` | 可选 | 到达后不可访问，但仍在数据库中 | 无强制限制 |
| `deletion_date` | **必填** | 到达后从数据库彻底删除 | **最大 31 天** |

### 2. 删除日期强制限制
[src/api/core/sends.rs#L145-L149](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L145-L149)

```rust
if data.deletion_date > Utc::now() + TimeDelta::try_days(31).unwrap() {
    err!(
        "You cannot have a Send with a deletion date that far into the future. \
        Adjust the Deletion Date to a value less than 31 days from now and try again."
    );
}
```

### 3. 定时清理任务

#### 触发配置
[src/config.rs#L543-L545](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/config.rs#L543-L545)
```rust
send_purge_schedule: String, false, def, "0 5 * * * *".to_owned();
// 默认: 每小时的第 5 分钟执行 (cron 表达式)
```

#### 任务调度
[src/main.rs#L679-L684](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/main.rs#L679-L684)
```rust
// Purge sends that are past their deletion date.
if !CONFIG.send_purge_schedule().is_empty() {
    sched.add(Job::new(CONFIG.send_purge_schedule().parse().unwrap(), || {
        runtime.spawn(api::purge_sends(pool.clone()));
    }));
}
```

#### 清理执行
[src/api/core/sends.rs#L61-L68](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/api/core/sends.rs#L61-L68)
```rust
pub async fn purge_sends(pool: DbPool) {
    debug!("Purging sends");
    if let Ok(conn) = pool.get().await {
        Send::purge(&conn).await;
    } else {
        error!("Failed to get DB connection while purging sends");
    }
}
```

#### 数据库查询
[src/db/models/send.rs#L329-L335](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/db/models/send.rs#L329-L335)
```rust
pub async fn find_by_past_deletion_date(conn: &DbConn) -> Vec<Self> {
    let now = Utc::now().naive_utc();
    conn.run(move |conn| {
        sends::table.filter(sends::deletion_date.lt(now)).load::<Self>(conn).expect("Error loading sends")
    })
    .await
}
```

#### 删除操作
[src/db/models/send.rs#L231-L243](file:///d:/fz/0601/solo-dogfeeding/code/7-vaultwarden/src/db/models/send.rs#L231-L243)
```rust
pub async fn delete(&self, conn: &DbConn) -> EmptyResult {
    self.update_users_revision(conn).await;

    // 如果是文件 Send，删除物理文件
    if self.atype == SendType::File as i32 {
        let operator = CONFIG.opendal_operator_for_path_type(&PathType::Sends)?;
        operator.delete_with(&self.uuid).recursive(true).await.ok();
    }

    // 删除数据库记录
    conn.run(move |conn| {
        diesel::delete(sends::table.filter(sends::uuid.eq(&self.uuid))).execute(conn).map_res("Error deleting send")
    })
    .await
}
```

---

## 十、策略总结

### 访问条件逻辑真值表

| Send 存在 | 未超限 | 未过期 | 未到删除时间 | 未禁用 | 密码正确 | 结果 |
|----------|--------|--------|-------------|--------|----------|------|
| ❌ | - | - | - | - | - | 404 |
| ✅ | ❌ | - | - | - | - | 404 |
| ✅ | ✅ | ❌ | - | - | - | 404 |
| ✅ | ✅ | ✅ | ❌ | - | - | 404 |
| ✅ | ✅ | ✅ | ✅ | ❌ | - | 404 |
| ✅ | ✅ | ✅ | ✅ | ✅ | 无密码 | ✅ 成功 |
| ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ 成功 |
| ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ 密码错误 |

### 关键设计决策

1. **双重时间控制**：`expiration_date` 用于访问控制，`deletion_date` 用于数据清理
2. **最大 31 天限制**：防止用户创建永久存在的 Send
3. **定时清理**：每小时检查一次，确保过期数据及时清理
4. **统一错误信息**：防止通过错误差异进行探测攻击
5. **文件计数时机**：获取下载链接时即计数，而非下载完成时
6. **下载时不二次检查**：获取链接后，有效期内可自由下载，不再验证 Send 状态
7. **双存储分支**：本地模式走 JWT + 后端转发，S3 模式走预签名直连
