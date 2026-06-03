# 事件日志与审计线索分析

本文档详细分析 Vaultwarden 项目中的事件日志（Event Log）和审计线索（Audit Trail）实现机制，包括用户事件、组织事件的分类，记录写入流程，以及操作者归属的设计。

---

## 1. 核心数据模型

### 1.1 Event 结构体

事件记录的核心数据结构定义在 [src/db/models/event.rs](src/db/models/event.rs#L26-L44)：

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

事件类型按数值范围分类，定义在 [src/db/models/event.rs](src/db/models/event.rs#L48-L142)：

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

## 2. 事件日志开关机制

### 2.1 配置项定义

事件日志功能由全局配置控制，定义在 [src/config.rs](src/config.rs#L629)：

```rust
/// Enable event logging |> Enables event logging for organizations.
org_events_enabled:     bool,   false,  def,    false;
```

- **默认值**：`false`（关闭）
- **配置方式**：环境变量 `ORG_EVENTS_ENABLED=true` 或 config.json 中设置

### 2.2 开关检查点（全链路覆盖）

`CONFIG.org_events_enabled()` 在以下 6 个关键位置进行检查，确保开关关闭时完全不产生事件记录：

| 检查位置 | 文件 | 作用 |
|----------|------|------|
| 记录入口1 | [src/api/core/events.rs:223](src/api/core/events.rs#L223) | `log_user_event()` 函数入口，用户事件第一道防线 |
| 记录入口2 | [src/api/core/events.rs:272](src/api/core/events.rs#L272) | `log_event()` 函数入口，组织事件第一道防线 |
| 客户端上报 | [src/api/core/events.rs:166](src/api/core/events.rs#L166) | `/events/collect` 端点入口，直接返回空成功 |
| 查询接口1 | [src/api/core/events.rs:41](src/api/core/events.rs#L41) | 组织事件查询，返回空数组避免客户端报错 |
| 查询接口2 | [src/api/core/events.rs:69](src/api/core/events.rs#L69) | 密码库事件查询，返回空数组 |
| 查询接口3 | [src/api/core/events.rs:104](src/api/core/events.rs#L104) | 成员事件查询，返回空数组 |

**开关检查代码示例**：
```rust
// 记录函数入口检查
pub async fn log_user_event(...) {
    if !CONFIG.org_events_enabled() {
        return;  // 直接返回，不执行任何后续逻辑
    }
    log_user_event_impl(...).await;
}

// 查询接口检查
let events_json: Vec<Value> = if CONFIG.org_events_enabled() {
    // 执行查询...
} else {
    Vec::with_capacity(0)  // 返回空数组，不报错
};
```

### 2.3 客户端感知开关

组织是否启用事件日志通过 `useEvents` 字段告知客户端，定义在 [src/db/models/organization.rs](src/db/models/organization.rs#L209)：

```rust
"useEvents": CONFIG.org_events_enabled(),
```

客户端据此决定是否启用本地事件收集和上报功能。

---

## 3. 事件记录写入机制

### 3.1 两大记录入口

事件记录通过两个独立的函数入口处理，定义在 [src/api/core/events.rs](src/api/core/events.rs#L222-L323)：

| 函数 | 适用场景 | 特点 |
|------|----------|------|
| `log_user_event()` | **用户类事件** (1000-1099) | 批量写入，每个用户所属组织各存一条 |
| `log_event()` | **组织类事件** (1100-1799) | 单条写入，只关联目标组织 |

### 3.2 用户事件记录流程 (log_user_event)

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

**实现流程：** [log_user_event_impl](src/api/core/events.rs#L229-L261)

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
// [src/api/identity.rs:117-124]
log_user_event(
    EventType::UserLoggedIn as i32,
    &user_id,
    client_header.device_type,
    &client_header.ip.ip,
    &conn,
).await;
```

### 3.3 组织事件记录流程 (log_event)

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

**实现流程：** [log_event_impl](src/api/core/events.rs#L279-L323)

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
// [src/api/core/organizations.rs:515-522]
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

### 3.4 数据库写入方法

| 方法 | 用途 | 特性 |
|------|------|------|
| `Event::save()` | 单条保存 | UPSERT 模式，冲突则更新 |
| `Event::save_user_event()` | 用户事件批量保存 | INSERT OR IGNORE，跳过重复 |

SQLite/MySQL/PostgreSQL 的写入差异在 [src/db/models/event.rs:203-256](src/db/models/event.rs#L203-L256) 中通过 `db_run!` 宏统一处理。

---

## 4. 客户端事件收集与日期处理

### 4.1 收集端点与数据结构

客户端（浏览器扩展、移动应用）通过 `/events/collect` 端点上报客户端侧事件，定义在 [src/api/core/events.rs:164-220](src/api/core/events.rs#L164-L220)：

```rust
#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
struct EventCollection {
    // 必选字段
    r#type: i32,      // 事件类型
    date: String,     // 客户端上报的事件时间 (RFC3339格式)

    // 可选字段
    cipher_id: Option<CipherId>,
    organization_id: Option<OrganizationId>,
}
```

### 4.2 客户端日期处理机制

**日期解析函数** 定义在 [src/util.rs:487-489](src/util.rs#L487-L489)：

```rust
pub fn parse_date(date: &str) -> NaiveDateTime {
    DateTime::parse_from_rfc3339(date).unwrap().naive_utc()
}
```

**处理流程**：
1. 客户端上报的 `date` 字段必须是 **RFC3339 格式**（如 `2024-01-15T10:30:00Z`）
2. 服务端调用 `parse_date()` 解析为 UTC 时间
3. 解析后的日期直接透传给 `log_user_event_impl` 或 `log_event_impl`
4. 通过 `Some(event_date)` 传入，覆盖默认的"当前时间"

**代码证据** - [src/api/core/events.rs:171-178](src/api/core/events.rs#L171-L178)：
```rust
for event in data.iter() {
    let event_date = parse_date(&event.date);  // 解析客户端日期
    match event.r#type {
        1000..=1099 => {
            log_user_event_impl(
                event.r#type,
                &headers.user.uuid,
                headers.device.atype,
                Some(event_date),  // 传入客户端日期，而非服务端当前时间
                &headers.ip.ip,
                &conn,
            ).await;
        }
        // ... 其他类型处理
    }
}
```

**设计意图**：
- 保留事件在**客户端实际发生**的时间，而非服务端接收时间
- 解决网络延迟、批量上报导致的时间偏差问题
- 确保审计时间线的准确性

### 4.3 客户端上报的事件类型路由

| 类型范围 | 处理方式 | 示例 |
|----------|----------|------|
| 1000-1099 | 调用 `log_user_event_impl()`，传入客户端日期 | UserClientExportedVault (1007) |
| 1600-1699 | 调用 `log_event_impl()`，以 organization_id 为源 | OrganizationClientExportedVault (1602) |
| 其他 (1100等) | 通过 cipher_id 查询所属组织，再调用 `log_event_impl()` | CipherClientViewed (1107) |

**客户端事件示例** - 查看密码字段：
```
客户端上报事件: { type: 1108, cipherId: "xxx", date: "2024-01-15T10:30:00Z" }
    ↓
服务端通过 cipherId 查询所属组织
    ↓
调用 log_event_impl，传入客户端日期 2024-01-15T10:30:00Z
```

---

## 5. 操作者归属与API输出

### 5.1 归属矩阵

| 事件类别 | user_uuid (主体) | act_user_uuid (操作者) | 示例场景 |
|----------|------------------|------------------------|----------|
| **用户事件** (1000-1099) | ✅ 目标用户 | ✅ 目标用户 (自操作) | 用户A登录、改密码 |
| **密码库事件** (1100-1199) | ❌ None | ✅ 修改者 | 管理员B修改了共享密码 |
| **集合事件** (1300-1399) | ❌ None | ✅ 创建/修改者 | 用户A创建集合 |
| **组事件** (1400-1499) | ❌ None | ✅ 创建/修改者 | 管理员B创建组 |
| **组织用户事件** (1500-1599) | ❌ None | ✅ 操作者 | 管理员B邀请用户C加入 |
| **组织事件** (1600-1699) | ❌ None | ✅ 修改者 | 所有者A修改组织设置 |

### 5.2 操作者字段的API输出映射

`act_user_uuid` 字段在 JSON 输出中映射为 `actingUserId`，定义在 [src/db/models/event.rs:172-193](src/db/models/event.rs#L172-L193)：

```rust
pub fn to_json(&self) -> Value {
    json!({
        "type": self.event_type,
        "userId": self.user_uuid,           // 事件主体 (可能为null)
        "actingUserId": self.act_user_uuid, // 操作者 (始终有值)
        "organizationId": self.org_uuid,
        "cipherId": self.cipher_uuid,
        "collectionId": self.collection_uuid,
        "groupId": self.group_uuid,
        "organizationUserId": self.org_user_uuid,
        "date": format_date(&self.event_date),
        "deviceType": self.device_type,
        "ipAddress": self.ip_address,
        "policyId": self.policy_uuid,
        // ... 其他字段
    })
}
```

**输出字段对照表**：

| 数据库字段 | JSON输出字段 | 说明 |
|------------|--------------|------|
| `user_uuid` | `userId` | 事件主体用户ID，非用户事件时为null |
| `act_user_uuid` | `actingUserId` | 实际执行者ID，始终有值 |
| `org_user_uuid` | `organizationUserId` | 组织成员关系ID |
| `org_uuid` | `organizationId` | 组织ID |

### 5.3 归属场景示例

**场景1：用户自己修改密码** (UserChangedPassword = 1001)
```
userId        = "用户A" (事件主体)
actingUserId  = "用户A" (操作者是自己)
organizationId = null / 所属组织ID
```
来源：[src/api/core/accounts.rs:400-401](src/api/core/accounts.rs#L400-L401)

**场景2：管理员重置成员密码** (OrganizationUserAdminResetPassword = 1508)
```
organizationUserId = "成员A的关系ID" (事件主体)
actingUserId       = "管理员B" (实际执行者)
organizationId     = "组织X"
userId             = null (注意：此字段为空！)
```

**场景3：管理员删除用户成员关系** (OrganizationUserRemoved = 1503)
```
organizationUserId = "被删除成员的关系ID"
actingUserId       = "执行删除的管理员"
organizationId     = "组织ID"
```
来源：[src/api/core/organizations.rs:1703-1706](src/api/core/organizations.rs#L1703-L1706)

**场景4：用户主动离开组织** (OrganizationUserLeft = 1516)
```
organizationUserId = "离开用户的关系ID"
actingUserId       = "离开的用户" (自操作)
organizationId     = "组织ID"
```
来源：[src/api/core/organizations.rs:270-273](src/api/core/organizations.rs#L270-L273)

---

## 6. 成员事件查询逻辑：审计边界修正

### 6.1 查询端点

所有查询接口定义在 [src/api/core/events.rs](src/api/core/events.rs#L33-L126)：

| 端点 | 权限 | 说明 |
|------|------|------|
| `GET /api/organizations/{org_id}/events` | 管理员 | 查询组织全部事件 |
| `GET /api/organizations/{org_id}/users/{member_id}/events` | 管理员 | 查询组织内某成员的事件 |
| `GET /api/ciphers/{cipher_id}/events` | 密码库管理员 | 查询某密码库的事件 |

### 6.2 关键区分：org_user_uuid 的角色

首先澄清一个重要概念：**`org_user_uuid`（API输出为 `organizationUserId`）只负责"记录"，不参与"查询过滤"**。

| 字段 | 作用 | 是否用于查询过滤 | 代码证据 |
|------|------|----------------|----------|
| `org_user_uuid` | 记录**目标成员是谁**（哪个成员关系被操作） | ❌ **不参与** | 存储时填充，查询WHERE条件中未使用 |
| `user_uuid` | 记录**主体用户是谁**（仅限用户类事件） | ✅ 参与过滤 | WHERE 条件第一部分 |
| `act_user_uuid` | 记录**操作者是谁** | ✅ 参与过滤 | WHERE 条件第二部分 |

**代码证据** - 组织用户事件记录时填充 org_user_uuid：
```rust
// 组织用户事件 (1500-1599): org_user_uuid = source_uuid
// [src/api/core/events.rs:306-308]
1500..=1599 => {
    event.org_user_uuid = Some(source_uuid.to_owned().into());
}
```

**但查询时 org_user_uuid 完全不参与过滤**，见下文分析。

### 6.3 成员事件查询：双字段过滤设计

**核心查询方法**：`find_by_org_and_member` 定义在 [src/db/models/event.rs:292-316](src/db/models/event.rs#L292-L316)

```rust
pub async fn find_by_org_and_member(
    org_uuid: &OrganizationId,
    member_uuid: &MembershipId,  // 输入：成员关系ID (注意：不是UserId)
    start: &NaiveDateTime,
    end: &NaiveDateTime,
    conn: &DbConn,
) -> Vec<Self> {
    conn.run(move |conn| {
        event::table
            // 步骤1：通过成员关系ID JOIN，找到对应的用户ID
            .inner_join(users_organizations::table
                .on(users_organizations::uuid.eq(member_uuid)))
            .filter(event::org_uuid.eq(org_uuid))
            .filter(event::event_date.between(start, end))
            // 步骤2：只用用户ID匹配两个字段，**不涉及 org_user_uuid**
            .filter(
                event::user_uuid
                    .eq(users_organizations::user_uuid.nullable())
                    .or(event::act_user_uuid.eq(users_organizations::user_uuid.nullable())),
            )
            .select(event::all_columns)
            .order_by(event::event_date.desc())
            .limit(Self::PAGE_SIZE)
            .load::<Self>(conn)
            .expect("Error filtering events")
    })
    .await
}
```

**SQL 逻辑拆解**：

```sql
SELECT event.* 
FROM event
INNER JOIN users_organizations 
    ON users_organizations.uuid = '输入的成员关系ID'  -- 目的：获取该成员关系对应的 user_uuid
WHERE event.org_uuid = '组织ID'
  AND event.event_date BETWEEN start AND end
  AND (
    -- 过滤条件1：该用户作为事件主体 (user_uuid 字段)
    event.user_uuid = users_organizations.user_uuid
    OR 
    -- 过滤条件2：该用户作为事件操作者 (act_user_uuid 字段)
    event.act_user_uuid = users_organizations.user_uuid
  )
-- 注意：WHERE 条件中 **完全没有** event.org_user_uuid 的判断！
ORDER BY event.event_date DESC
LIMIT 30
```

### 6.4 审计边界：实际覆盖范围

| 匹配条件 | 字段 | 覆盖的事件类型 | 示例事件 |
|----------|------|--------------|----------|
| 条件1 | `event.user_uuid = 用户ID` | 用户类事件 (1000-1099) | 用户登录、改密码 |
| 条件2 | `event.act_user_uuid = 用户ID` | 所有类型事件 | 用户创建集合、邀请成员、批准设备等 |

**⚠️ 重要边界说明**：

对于**组织用户事件**（如 OrganizationUserInvited = 1500）：
- 记录时：`org_user_uuid = 被邀请成员的关系ID`，`act_user_uuid = 邀请者用户ID`
- 查询时：过滤条件**不看 org_user_uuid**，只看 `user_uuid` 和 `act_user_uuid`
- 结果：当管理员A邀请成员B加入时：
  - 查询**成员B**的事件：能查到吗？→ 看B的 `act_user_uuid` 有没有匹配（B是被邀请者，不是操作者，所以查不到）
  - 查询**管理员A**的事件：能查到吗？→ 能，因为A的 `act_user_uuid` 匹配
  - ❌ **成员B的审计列表看不到"自己被邀请"这个事件**

**查询逻辑的设计意图**：
- 以"用户身份"为中心，追踪该用户**做了什么**（act_user_uuid）和**什么事情发生在他身上**（user_uuid）
- 不是以"成员关系"为中心，不追踪"针对该成员关系的所有操作"
- org_user_uuid 仅用于事件详情展示（告诉客户端这个事件是针对哪个成员关系的），不用于过滤

### 6.5 字段角色总结表

| 字段 | 存储时 | 查询过滤时 | API输出字段 |
|------|--------|-----------|------------|
| `org_user_uuid` | 填充目标成员关系ID | ❌ 不参与 | `organizationUserId`（仅展示用） |
| `user_uuid` | 填充主体用户ID（仅限用户事件） | ✅ 条件1 | `userId` |
| `act_user_uuid` | 始终填充操作者用户ID | ✅ 条件2 | `actingUserId` |

**一句话总结**：
> `organizationUserId` 是事件的"描述信息"，告诉你这个事件影响了谁；但查询过滤时，系统只认 `userId` 和 `actingUserId` 这两个用户身份字段。

---

## 7. 事件清理机制

### 7.1 保留天数配置

配置项定义在 [src/config.rs:658-659](src/config.rs#L658-L659)：

```rust
/// Events days retain |> Number of days to retain events stored in the database. If unset, events are kept indefinitely.
events_days_retain:     i64,    false,   option;
```

- 未设置时：事件永久保留
- 设置为 N 时：仅保留最近 N 天的事件

### 7.2 清理任务

清理任务定义在 [src/api/core/events.rs:325-337](src/api/core/events.rs#L325-L337)，由主程序调度：

```rust
pub async fn event_cleanup_job(pool: DbPool) {
    debug!("Start events cleanup job");
    if CONFIG.events_days_retain().is_none() {
        debug!("events_days_retain is not configured, abort");
        return;
    }

    if let Ok(conn) = pool.get().await {
        Event::clean_events(&conn).await.ok();
    } else {
        error!("Failed to get DB connection while trying to cleanup the events table");
    }
}
```

清理方法定义在 [src/db/models/event.rs:336-348](src/db/models/event.rs#L336-L348)：

```rust
pub async fn clean_events(conn: &DbConn) -> EmptyResult {
    if let Some(days_to_retain) = CONFIG.events_days_retain() {
        let dt = Utc::now().naive_utc() - TimeDelta::try_days(days_to_retain).unwrap();
        conn.run(move |conn| {
            diesel::delete(event::table.filter(event::event_date.lt(dt)))
                .execute(conn)
                .map_res("Error cleaning old events")
        })
        .await
    } else {
        Ok(())
    }
}
```

---

## 8. 事件记录触发点汇总

### 8.1 用户事件触发点 (1000-1099)

| 事件 | 触发文件/位置 | 说明 |
|------|--------------|------|
| UserLoggedIn (1000) | [src/api/identity.rs:117](src/api/identity.rs#L117) | 登录成功 |
| UserFailedLogIn (1005) | [src/api/identity.rs:128](src/api/identity.rs#L128) | 登录失败 |
| UserChangedPassword (1001) | [src/api/core/accounts.rs:400](src/api/core/accounts.rs#L400) | 修改主密码 |
| UserRecovered2fa (1004) | [src/api/identity.rs:870](src/api/identity.rs#L870) | 恢复2FA |
| UserRequestedDeviceApproval (1010) | [src/api/core/accounts.rs:1500](src/api/core/accounts.rs#L1500) | 请求设备批准 |

### 8.2 组织事件触发点 (1500-1599)

| 事件 | 触发文件/位置 | 说明 |
|------|--------------|------|
| OrganizationUserInvited (1500) | [src/api/core/organizations.rs:1133](src/api/core/organizations.rs#L1133) | 邀请成员 |
| OrganizationUserConfirmed (1501) | [src/api/core/organizations.rs:1429](src/api/core/organizations.rs#L1429) | 成员确认加入 |
| OrganizationUserRemoved (1503) | [src/api/core/organizations.rs:1703](src/api/core/organizations.rs#L1703) | 移除成员 |
| OrganizationUserLeft (1516) | [src/api/core/organizations.rs:270](src/api/core/organizations.rs#L270) | 用户主动离开 |
| OrganizationUserApprovedAuthRequest (1513) | [src/api/core/accounts.rs:1594](src/api/core/accounts.rs#L1594) | 批准设备登录 |
| OrganizationUserRejectedAuthRequest (1514) | [src/api/core/accounts.rs:1605](src/api/core/accounts.rs#L1605) | 拒绝设备登录 |

---

## 9. 总结

### 9.1 设计核心原则

1. **双轨记录**：用户事件采用"全局+组织副本"双轨模式，确保审计完整性
2. **角色分离**：`user_uuid`（主体）与 `act_user_uuid`（操作者）分离，支持管理员操作审计
3. **类型驱动**：通过事件类型数值范围自动路由到不同处理逻辑
4. **上下文完整**：每条事件记录设备类型和IP地址，便于安全分析
5. **全链路开关**：`org_events_enabled` 在所有入口点检查，关闭时完全无性能损耗
6. **客户端时间保留**：上报事件使用客户端日期，确保审计时间线准确
7. **⚠️ 成员查询边界**：成员接口只覆盖 `user_uuid` 和 `act_user_uuid` 身份轨迹，**不覆盖** `organizationUserId` 目标成员维度

### 9.2 审计查询建议

| 审计目标 | 查询方式 | 关键字段 | 覆盖范围说明 |
|----------|----------|----------|-------------|
| 用户主动操作历史 | 按 `act_user_uuid` 查询 | `actingUserId`, `event_date` | ✅ 该用户执行的所有操作 |
| 用户被操作历史 | 按 `user_uuid` 查询 | `userId`, `event_date` | ✅ 该用户作为主体的用户类事件（登录、改密码等） |
| ⚠️ 针对某成员的操作历史 | **不能**通过成员查询接口，需直接查 `org_user_uuid` | `organizationUserId` | ❌ 成员查询接口不覆盖此字段，需自行SQL查询 |
| 组织安全审计 | 按 `org_uuid` 分页查询 | `organizationId`, `event_type` | ✅ 组织内所有事件 |
| 敏感操作追踪 | 按 `event_type` 过滤特定事件 | `type`, `ipAddress` | ✅ 所有类型的特定事件 |
| ⚠️ 指定成员轨迹查询 | 调用成员事件查询接口 | 内部匹配 `user_uuid` OR `act_user_uuid` | ✅ 该用户做了什么 + 该用户作为主体的用户事件<br>❌ **不包含** 针对该成员的组织用户操作（如被邀请、被移除） |

### 9.3 成员接口覆盖边界详解

**代码证据**：[src/db/models/event.rs:292-316](src/db/models/event.rs#L292-L316)

```rust
// 成员事件查询的WHERE条件：
.filter(
    event::user_uuid                         // 条件1：用户身份（主体）
        .eq(users_organizations::user_uuid.nullable())
        .or(event::act_user_uuid              // 条件2：用户身份（操作者）
            .eq(users_organizations::user_uuid.nullable())),
)
// 注意：没有 event.org_user_uuid 的判断！
```

**覆盖矩阵**：

| 事件场景 | 记录字段 | 成员接口能查到吗？ | 原因 |
|----------|----------|-------------------|------|
| 用户A自己登录 | `user_uuid=A` | ✅ 能 | 匹配条件1 |
| 用户A改密码 | `user_uuid=A`, `act_user_uuid=A` | ✅ 能 | 匹配条件1或2 |
| 管理员B邀请用户A加入 | `org_user_uuid=A的成员关系ID`, `act_user_uuid=B` | ❌ 查A：不能<br>✅ 查B：能 | A既不是user_uuid也不是act_user_uuid |
| 管理员B移除用户A | `org_user_uuid=A的成员关系ID`, `act_user_uuid=B` | ❌ 查A：不能<br>✅ 查B：能 | 同上 |
| 用户A主动离开组织 | `org_user_uuid=A的成员关系ID`, `act_user_uuid=A` | ✅ 能 | 匹配条件2（act_user_uuid=A） |
| 用户A创建集合 | `act_user_uuid=A` | ✅ 能 | 匹配条件2 |

**关键结论**：
> 成员查询接口是以"**用户身份**"为中心的活动追踪，不是以"**成员关系**"为中心的操作追踪。
> 如果需要审计"针对某成员的所有操作"（如谁邀请了他、谁移除了他），必须直接查询 `org_user_uuid` 字段，当前API不提供此能力。

### 9.4 代码溯源路径

```
配置定义:
  src/config.rs (org_events_enabled, events_days_retain)

数据模型:
  src/db/models/event.rs (Event 结构体, EventType 枚举, 查询方法)
    - 注意 find_by_org_and_member 只过滤 user_uuid 和 act_user_uuid

记录入口:
  src/api/core/events.rs (log_user_event, log_event, 客户端收集)
    - log_user_event: 填充 user_uuid 和 act_user_uuid
    - log_event: 组织用户事件填充 org_user_uuid 和 act_user_uuid，不填充 user_uuid

事件触发点:
  用户类:     src/api/identity.rs, src/api/core/accounts.rs
  组织类:     src/api/core/organizations.rs (邀请/移除成员等事件填充 org_user_uuid)
  密码库类:   src/api/core/ciphers.rs
  2FA类:      src/api/core/two_factor/*.rs

查询API:
  src/api/core/events.rs
    - get_org_events: 组织全部事件
    - get_user_events: ⚠️  只覆盖 user_uuid 和 act_user_uuid
    - get_cipher_events: 密码库事件

工具函数:
  src/util.rs (parse_date)
```

### 9.5 审计工作流程图

```
记录阶段:
  事件触发 → log_user_event/log_event → 填充各字段 → 写入DB
    用户类事件: user_uuid=用户, act_user_uuid=用户
    组织用户事件: org_user_uuid=成员关系, act_user_uuid=操作者 (user_uuid=None)

查询阶段 (成员接口):
  输入 member_id → JOIN users_organizations → 得到 user_id
    → WHERE user_uuid = user_id OR act_user_uuid = user_id
    → 返回结果

查询阶段 (缺失能力):
  若需查"针对该成员的操作" → 需 WHERE org_user_uuid = member_id
  → 当前无此API，需自行实现
```
