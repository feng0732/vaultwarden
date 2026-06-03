# 事件日志与审计线索分析

本文档详细分析 Vaultwarden 项目中的事件日志（Event Log）和审计线索（Audit Trail）实现机制，包括用户事件、组织事件的分类，记录写入流程，以及操作者归属的设计。

---

## 1. 核心数据模型

### 1.1 Event 结构体

事件记录的核心数据结构定义在 [event.rs](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/db/models/event.rs#L26-L44)：

```rust
pub struct Event {
    pub uuid: EventId,                    // 事件唯一ID
    pub event_type: i32,                  // 事件类型 (EventType 枚举值)
    pub user_uuid: Option<UserId>,        // 目标用户ID（事件发生在谁身上）
    pub org_uuid: Option<OrganizationId>, // 关联组织ID
    pub cipher_uuid: Option<CipherId>,    // 关联密码库ID
    pub collection_uuid: Option<CollectionId>, // 关联集合ID
    pub group_uuid: Option<GroupId>,      // 关联组ID
    pub org_user_uuid: Option<MembershipId>, // 组织成员关系ID
    pub act_user_uuid: Option<UserId>,    // 操作者ID（谁执行了这个动作）
    pub device_type: Option<i32>,         // 设备类型
    pub ip_address: Option<String>,       // IP地址
    pub event_date: NaiveDateTime,        // 事件时间
    pub policy_uuid: Option<OrgPolicyId>, // 策略ID
    pub provider_uuid: Option<String>,    // 服务商ID (预留)
    pub provider_user_uuid: Option<String>, // 服务商用户ID (预留)
    pub provider_org_uuid: Option<String>,  // 服务商组织ID (预留)
}
```

**关键字段解读：**

| 字段 | 说明 | 归属关系 |
|------|------|----------|
| `user_uuid` | **事件主体** - 事件发生在哪个用户身上。例如：用户A登录时，此字段为用户A的ID | 被动接受者 |
| `act_user_uuid` | **操作者** - 实际执行动作的用户。例如：管理员B重置了用户A的密码，`act_user_uuid` 是B的ID | 主动执行者 |
| `org_user_uuid` | 组织成员关系ID，关联 `users_organizations` 表 | 成员关系标识 |
| `org_uuid` | 事件所属的组织 | 归属组织 |

### 1.2 EventType 事件类型枚举

事件类型按数值范围分类，定义在 [event.rs](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/db/models/event.rs#L48-L142)：

| 数值范围 | 类别 | 示例事件 |
|----------|------|----------|
| 1000-1099 | **用户事件** | UserLoggedIn, UserChangedPassword, UserFailedLogIn |
| 1100-1199 | **密码库事件** | CipherCreated, CipherUpdated, CipherDeleted |
| 1300-1399 | **集合事件** | CollectionCreated, CollectionUpdated |
| 1400-1499 | **组事件** | GroupCreated, GroupUpdated |
| 1500-1599 | **组织用户事件** | OrganizationUserInvited, OrganizationUserRemoved |
| 1600-1699 | **组织事件** | OrganizationUpdated, OrganizationPurgedVault |
| 1700-1799 | **策略事件** | PolicyUpdated |

主要事件类型：
```rust
pub enum EventType {
    // 用户事件 (1000-1099)
    UserLoggedIn = 1000,
    UserChangedPassword = 1001,
    UserUpdated2fa = 1002,
    UserDisabled2fa = 1003,
    UserRecovered2fa = 1004,
    UserFailedLogIn = 1005,
    UserFailedLogIn2fa = 1006,
    UserClientExportedVault = 1007,
    UserRequestedDeviceApproval = 1010,

    // 密码库事件 (1100-1199)
    CipherCreated = 1100,
    CipherUpdated = 1101,
    CipherDeleted = 1102,
    CipherShared = 1105,
    CipherSoftDeleted = 1115,
    CipherRestored = 1116,

    // 集合事件 (1300-1399)
    CollectionCreated = 1300,
    CollectionUpdated = 1301,
    CollectionDeleted = 1302,

    // 组事件 (1400-1499)
    GroupCreated = 1400,
    GroupUpdated = 1401,
    GroupDeleted = 1402,

    // 组织用户事件 (1500-1599)
    OrganizationUserInvited = 1500,
    OrganizationUserConfirmed = 1501,
    OrganizationUserRemoved = 1503,
    OrganizationUserRevoked = 1511,
    OrganizationUserLeft = 1516,

    // 组织事件 (1600-1699)
    OrganizationUpdated = 1600,
    OrganizationPurgedVault = 1601,

    // 策略事件 (1700-1799)
    PolicyUpdated = 1700,
}
```

---

## 2. 事件记录写入机制

### 2.1 两大记录入口

事件记录通过两个独立的函数入口处理，定义在 [events.rs](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/events.rs#L222-L323)：

| 函数 | 适用场景 | 特点 |
|------|----------|------|
| `log_user_event()` | **用户类事件** (1000-1099) | 批量写入，每个用户所属组织各存一条 |
| `log_event()` | **组织类事件** (1100-1799) | 单条写入，只关联目标组织 |

### 2.2 用户事件记录流程 (log_user_event)

**函数签名：**
```rust
pub async fn log_user_event(
    event_type: i32,        // 事件类型 (1000-1099)
    user_id: &UserId,       // 目标用户
    device_type: i32,       // 设备类型
    ip: &IpAddr,            // IP地址
    conn: &DbConn           // 数据库连接
)
```

**实现流程：** [log_user_event_impl](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/events.rs#L229-L261)

```
用户事件触发 (如登录成功)
    ↓
查询用户的所有已确认组织成员关系 (Membership::find_confirmed_by_user)
    ↓
创建事件列表 (memberships.len() + 1 条)
    ├─ 第1条: 无组织关联的全局用户事件
    │   ├─ user_uuid = 用户ID
    │   ├─ act_user_uuid = 用户ID (自我操作)
    │   ├─ org_uuid = None
    │   └─ 记录设备/IP
    │
    └─ 每条成员关系创建一条组织内事件
        ├─ user_uuid = 用户ID
        ├─ act_user_uuid = 用户ID
        ├─ org_uuid = 成员所属组织ID
        ├─ org_user_uuid = 成员关系ID
        └─ 记录设备/IP
    ↓
批量写入数据库 (Event::save_user_event)
```

**关键设计说明：**

1. **双重复制**：用户事件既保存全局记录（无组织），也在每个所属组织中保存副本
2. **自引用**：用户事件中 `user_uuid` 和 `act_user_uuid` 始终相同（用户自己操作自己）
3. **幂等写入**：使用 `insert_or_ignore_into` 避免重复记录错误

**代码示例** - 登录成功记录：
```rust
// [identity.rs:117-124]
log_user_event(
    EventType::UserLoggedIn as i32,
    &user_id,
    client_header.device_type,
    &client_header.ip.ip,
    &conn,
).await;
```

### 2.3 组织事件记录流程 (log_event)

**函数签名：**
```rust
pub async fn log_event(
    event_type: i32,            // 事件类型 (1100-1799)
    source_uuid: &str,          // 源对象ID (cipher/collection/group等)
    org_id: &OrganizationId,    // 所属组织ID
    act_user_id: &UserId,       // 操作者ID
    device_type: i32,           // 设备类型
    ip: &IpAddr,                // IP地址
    conn: &DbConn               // 数据库连接
)
```

**实现流程：** [log_event_impl](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/events.rs#L279-L323)

```
组织事件触发 (如创建集合)
    ↓
创建空 Event 对象
    ↓
按事件类型分类填充 source_uuid:
    ├─ 1100-1199 (密码库): cipher_uuid = source_uuid
    ├─ 1300-1399 (集合): collection_uuid = source_uuid
    ├─ 1400-1499 (组): group_uuid = source_uuid
    ├─ 1500-1599 (成员): org_user_uuid = source_uuid
    └─ 1700-1799 (策略): policy_uuid = source_uuid
    ↓
设置通用字段:
    ├─ org_uuid = 组织ID
    ├─ act_user_uuid = 操作者ID
    ├─ device_type / ip_address
    └─ user_uuid = None (非用户事件)
    ↓
单条写入数据库 (event.save)
```

**关键设计说明：**

1. **无 user_uuid**：组织类事件不设置 `user_uuid` 字段，事件主体是组织对象而非用户
2. **act_user 必填**：所有组织事件都必须指定操作者
3. **单条写入**：每条组织事件只写入一条记录，关联到对应的组织

**代码示例** - 创建集合：
```rust
// [organizations.rs:515-522]
log_event(
    EventType::CollectionCreated as i32,
    &collection.uuid,       // source_uuid = 集合ID
    &org_id,                // 组织ID
    &headers.user.uuid,     // 操作者 = 当前登录用户
    headers.device.atype,
    &headers.ip.ip,
    &conn,
).await;
```

### 2.4 数据库写入方法

| 方法 | 用途 | 特性 |
|------|------|------|
| `Event::save()` | 单条保存 | UPSERT 模式，冲突则更新 |
| `Event::save_user_event()` | 用户事件批量保存 | INSERT OR IGNORE，跳过重复 |

SQLite/MySQL/PostgreSQL 的写入差异在 [event.rs:203-256](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/db/models/event.rs#L203-L256) 中通过 `db_run!` 宏统一处理。

---

## 3. 操作者归属详解

### 3.1 归属矩阵

| 事件类别 | user_uuid (主体) | act_user_uuid (操作者) | 示例场景 |
|----------|------------------|------------------------|----------|
| **用户事件** (1000-1099) | ✅ 目标用户 | ✅ 目标用户 (自操作) | 用户A登录、改密码 |
| **密码库事件** (1100-1199) | ❌ None | ✅ 修改者 | 管理员B修改了共享密码 |
| **集合事件** (1300-1399) | ❌ None | ✅ 创建/修改者 | 用户A创建集合 |
| **组事件** (1400-1499) | ❌ None | ✅ 创建/修改者 | 管理员B创建组 |
| **组织用户事件** (1500-1599) | ❌ None | ✅ 操作者 | 管理员B邀请用户C加入 |
| **组织事件** (1600-1699) | ❌ None | ✅ 修改者 | 所有者A修改组织设置 |

### 3.2 归属场景示例

**场景1：用户自己修改密码** (UserChangedPassword = 1001)
```
user_uuid    = 用户A (目标主体)
act_user_uuid = 用户A (操作者是自己)
org_uuid     = None / 用户A所属组织
```
来源：[accounts.rs:400-401](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/accounts.rs#L400-L401)

**场景2：管理员重置成员密码** (OrganizationUserAdminResetPassword = 1508)
```
org_user_uuid = 成员A的组织关系ID (目标主体)
act_user_uuid = 管理员B (实际执行者)
org_uuid      = 组织X
user_uuid     = None (注意：此字段为空！)
```

**场景3：管理员删除用户成员关系** (OrganizationUserRemoved = 1503)
```
org_user_uuid = 被删除成员的关系ID
act_user_uuid = 执行删除的管理员
org_uuid      = 组织ID
```
来源：[organizations.rs:1703-1706](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/organizations.rs#L1703-L1706)

**场景4：用户主动离开组织** (OrganizationUserLeft = 1516)
```
org_user_uuid = 离开用户的关系ID
act_user_uuid = 离开的用户 (自操作)
org_uuid      = 组织ID
```
来源：[organizations.rs:270-273](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/organizations.rs#L270-L273)

---

## 4. 客户端事件收集

### 4.1 收集端点

客户端（浏览器扩展、移动应用）通过 `/events/collect` 端点上报客户端侧事件：

```rust
// [events.rs:164-220]
#[post("/collect", format = "application/json", data = "<data>")]
async fn post_events_collect(
    data: Json<Vec<EventCollection>>, 
    headers: Headers, 
    conn: DbConn
) -> EmptyResult
```

### 4.2 客户端上报的事件类型

| 类型范围 | 处理方式 | 示例 |
|----------|----------|------|
| 1000-1099 | 调用 `log_user_event()` | UserClientExportedVault (1007) |
| 1600-1699 | 调用 `log_event()`，以组织为源 | OrganizationClientExportedVault (1602) |
| 其他 (1100等) | 通过 cipher_id 关联到组织 | CipherClientViewed (1107) |

**客户端事件示例** - 查看密码字段：
```
客户端上报事件: { type: 1108, cipherId: "xxx", date: "..." }
    ↓
服务端通过 cipherId 查询所属组织
    ↓
调用 log_event 写入组织事件日志
```

---

## 5. 事件查询接口

### 5.1 查询端点

所有查询接口定义在 [events.rs](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/events.rs#L33-L126)：

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /api/organizations/{org_id}/events` | 管理员 | 查询组织全部事件 |
| `GET /api/organizations/{org_id}/users/{member_id}/events` | 管理员 | 查询组织内某成员的事件 |
| `GET /api/ciphers/{cipher_id}/events` | 密码库管理员 | 查询某密码库的事件 |

### 5.2 成员事件查询逻辑

**关键点：** 查询成员事件时，同时匹配 `user_uuid` 和 `act_user_uuid`

```sql
-- [event.rs:292-316] find_by_org_and_member
SELECT event.* 
FROM event
INNER JOIN users_organizations 
    ON users_organizations.uuid = member_uuid
WHERE event.org_uuid = org_uuid
  AND event.event_date BETWEEN start AND end
  AND (
    event.user_uuid = users_organizations.user_uuid   -- 作为事件主体
    OR 
    event.act_user_uuid = users_organizations.user_uuid  -- 作为操作者
  )
```

**设计意图：** 成员事件审计需同时包含：
1. 该用户作为**主体**被操作的事件（如被管理员重置密码）
2. 该用户作为**操作者**执行的事件（如用户修改了共享密码）

---

## 6. 事件清理机制

### 6.1 保留天数配置

通过 `CONFIG.events_days_retain()` 配置事件保留天数。

### 6.2 清理任务

```rust
// [events.rs:325-337] event_cleanup_job
pub async fn event_cleanup_job(pool: DbPool) {
    if let Some(days_to_retain) = CONFIG.events_days_retain() {
        let dt = Utc::now().naive_utc() - TimeDelta::try_days(days_to_retain).unwrap();
        diesel::delete(event::table.filter(event::event_date.lt(dt)))
            .execute(conn)
    }
}
```

---

## 7. 事件记录触发点汇总

### 7.1 用户事件触发点 (1000-1099)

| 事件 | 触发文件/位置 | 说明 |
|------|--------------|------|
| UserLoggedIn (1000) | [identity.rs:117](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/identity.rs#L117) | 登录成功 |
| UserFailedLogIn (1005) | [identity.rs:128](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/identity.rs#L128) | 登录失败 |
| UserChangedPassword (1001) | [accounts.rs:400](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/accounts.rs#L400) | 修改主密码 |
| UserRecovered2fa (1004) | [identity.rs:870](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/identity.rs#L870) | 恢复2FA |
| UserRequestedDeviceApproval (1010) | [accounts.rs:1500](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/accounts.rs#L1500) | 请求设备批准 |

### 7.2 组织事件触发点 (1500-1599)

| 事件 | 触发文件/位置 | 说明 |
|------|--------------|------|
| OrganizationUserInvited (1500) | [organizations.rs:1133](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/organizations.rs#L1133) | 邀请成员 |
| OrganizationUserConfirmed (1501) | [organizations.rs:1429](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/organizations.rs#L1429) | 成员确认加入 |
| OrganizationUserRemoved (1503) | [organizations.rs:1703](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/organizations.rs#L1703) | 移除成员 |
| OrganizationUserLeft (1516) | [organizations.rs:270](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/organizations.rs#L270) | 用户主动离开 |
| OrganizationUserApprovedAuthRequest (1513) | [accounts.rs:1594](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/accounts.rs#L1594) | 批准设备登录 |
| OrganizationUserRejectedAuthRequest (1514) | [accounts.rs:1605](file:///d:/fz/0601/solo-dogfeeding/code/13-vaultwarden/src/api/core/accounts.rs#L1605) | 拒绝设备登录 |

---

## 8. 总结

### 8.1 设计核心原则

1. **双轨记录**：用户事件采用"全局+组织副本"双轨模式，确保审计完整性
2. **角色分离**：`user_uuid`（主体）与 `act_user_uuid`（操作者）分离，支持管理员操作审计
3. **类型驱动**：通过事件类型数值范围自动路由到不同处理逻辑
4. **上下文完整**：每条事件记录设备类型和IP地址，便于安全分析

### 8.2 审计查询建议

| 审计目标 | 查询方式 | 关键字段 |
|----------|----------|----------|
| 用户所有操作 | 按 `act_user_uuid` 查询 | act_user_uuid, event_date |
| 用户被操作历史 | 按 `user_uuid` 或 `org_user_uuid` 查询 | user_uuid, org_user_uuid |
| 组织安全审计 | 按 `org_uuid` 分页查询 | org_uuid, event_type |
| 敏感操作追踪 | 按 `event_type` 过滤特定事件 | event_type, ip_address |

### 8.3 代码溯源路径

```
事件枚举定义: src/db/models/event.rs (EventType)
事件记录入口: src/api/core/events.rs (log_user_event, log_event)
事件触发点: 
  - 用户类: src/api/identity.rs, src/api/core/accounts.rs
  - 组织类: src/api/core/organizations.rs
  - 密码库: src/api/core/ciphers.rs
  - 2FA类: src/api/core/two_factor/*.rs
事件查询API: src/api/core/events.rs (get_org_events, get_user_events)
```
