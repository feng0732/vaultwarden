# 组织分组与访问继承机制详解

## 一、核心数据模型

### 1.1 实体关系图

```
Organization (组织)
├── Group (组)
│   ├── GroupUser (组-成员关联)
│   └── CollectionGroup (组-集合关联)
├── Membership (组织成员)
│   └── CollectionUser (成员-集合直接关联)
└── Collection (集合)
```

### 1.2 关键模型定义

#### Group (组)
- [group.rs: L18-L30](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L18-L30)
- 核心字段：
  - `access_all`: 组内成员是否可以访问组织内所有集合

#### GroupUser (组-成员关联)
- [group.rs: L43-L49](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L43-L49)
- 关联 `groups_uuid` (组ID) 和 `users_organizations_uuid` (成员ID)
- 注意：关联的是成员ID（MembershipId），不是用户ID（UserId）

#### CollectionGroup (组-集合关联)
- [group.rs: L32-L41](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L32-L41)
- 权限字段：
  - `read_only`: 只读
  - `hide_passwords`: 隐藏密码
  - `manage`: 管理权限

#### CollectionUser (直接授权)
- [collection.rs: L35-L44](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L35-L44)
- 用户直接与集合建立关联，跳过组层级

---

## 二、成员关系：GroupUser 的绑定机制

### 2.1 关联表结构

`groups_users` 表的复合主键是 `(groups_uuid, users_organizations_uuid)`，这意味着：

- 一个成员可以加入多个组
- 一个组可以包含多个成员
- 但同一个成员在同一个组中只能有一条记录

### 2.2 成员入组流程

**API 入口**：[organizations.rs: L2611-L2650](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/api/core/organizations.rs#L2611-L2650) (`add_update_group` 函数)

```rust
// 新增/更新组时的成员绑定
for assigned_member in members {
    let mut user_entry = GroupUser::new(group.uuid.clone(), assigned_member.clone());
    user_entry.save(conn).await?;
    // 触发用户数据版本更新
}
```

### 2.3 更新组时的全量替换机制

**关键逻辑**：[organizations.rs: L2594-L2595](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/api/core/organizations.rs#L2594-L2595)

```rust
// 更新组时，先删除所有旧的关联，再重新添加
CollectionGroup::delete_all_by_group(&group_id, &org_id, &conn).await?;
GroupUser::delete_all_by_group(&group_id, &org_id, &conn).await?;
// 然后调用 add_update_group 重新添加
```

**重要提示**：这是一个 **全量替换** 操作，不是增量更新。前端需要传递完整的成员列表。

---

## 三、访问授权：直接授权 vs 组继承

### 3.1 权限判定优先级

访问控制的核心判定在 [collection.rs: L149-L156](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L149-L156) (`can_access_collection` 函数)：

```rust
pub async fn can_access_collection(member: &Membership, col_id: &CollectionId, conn: &DbConn) -> bool {
    member.has_status(MembershipStatus::Confirmed)
        && (member.has_full_access()  // 1. 成员全局访问权限
            || CollectionUser::has_access_to_collection_by_user(...)  // 2. 直接授权
            || (CONFIG.org_groups_enabled()
                && (GroupUser::has_full_access_by_member(...)  // 3. 所在组有 access_all
                    || GroupUser::has_access_to_collection_by_member(...))))  // 4. 所在组关联了集合
}
```

**判定顺序**（OR 逻辑，任一满足即可）：
1. **成员全局权限**：`membership.access_all = true` 或用户是 Owner/Admin
2. **直接授权**：`users_collections` 表存在记录
3. **组级全局权限**：用户所在任一 `group.access_all = true`
4. **组级集合权限**：用户所在任一 group 与该 collection 有关联

### 3.2 组级权限判定

#### 3.2.1 组全局访问 (access_all)

**判定函数**：[group.rs: L593-L610](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L593-L610) (`has_full_access_by_member`)

```sql
SELECT COUNT(*) > 0
FROM groups_users
INNER JOIN groups ON groups.uuid = groups_users.groups_uuid
WHERE groups.organizations_uuid = ?org_uuid
  AND groups.access_all = true
  AND groups_users.users_organizations_uuid = ?member_uuid
```

只要成员所在的**任意一个组**设置了 `access_all = true`，该成员就拥有对组织所有集合的访问权。

#### 3.2.2 组集合关联访问

**判定函数**：[group.rs: L569-L591](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L569-L591) (`has_access_to_collection_by_member`)

```sql
SELECT COUNT(*) > 0
FROM groups_users
INNER JOIN collections_groups ON collections_groups.groups_uuid = groups_users.groups_uuid
INNER JOIN groups ON groups.uuid = groups_users.groups_uuid
INNER JOIN collections ON collections.uuid = collections_groups.collections_uuid
WHERE collections_groups.collections_uuid = ?collection_uuid
  AND groups_users.users_organizations_uuid = ?member_uuid
```

只要成员所在的**任意一个组**与目标集合有关联，该成员就获得该集合的访问权。

### 3.3 直接授权 vs 组继承的差异

| 维度 | 直接授权 (CollectionUser) | 组继承 (CollectionGroup) |
|------|--------------------------|-------------------------|
| 关联表 | `users_collections` | `collections_groups` |
| 关联键 | user_uuid + collection_uuid | group_uuid + collection_uuid |
| 权限传递 | 一对一 | 一对多（组内所有成员） |
| 更新影响 | 仅影响单个用户 | 影响组内所有用户 |
| 查询性能 | 直接查询 | 需多表 JOIN |

**代码位置对比**：
- 直接授权查询：[collection.rs: L854-L856](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L854-L856)
- 组继承查询：[collection.rs: L569-L591](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L569-L591)

---

## 四、继承规则：权限合并与计算

### 4.1 用户集合列表的获取

**完整查询逻辑**：[collection.rs: L225-L301](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L225-L301) (`find_by_user_uuid`)

用户可访问的集合来源有四个（OR 关系）：

```sql
WHERE 
  -- 1. 直接授权
  users_collections.user_uuid = ?user_uuid
  OR
  -- 2. 成员全局访问权限
  users_organizations.access_all = true
  OR
  -- 3. 组全局访问权限
  groups.access_all = true
  OR
  -- 4. 组集合关联
  (groups_users.users_organizations_uuid = users_organizations.uuid 
   AND collections_groups.collections_uuid IS NOT NULL)
```

### 4.2 权限字段的合并规则

当用户通过多个途径获得同一集合的访问权时，权限字段的判定遵循 **"最宽松原则"**：

以 `is_writable_by_user` 为例 [collection.rs: L426-L504](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L426-L504)：

```sql
WHERE
  -- 任一条件满足即可写
  users_organizations.atype <= Admin  -- 是 Owner/Admin
  OR users_organizations.access_all = true  -- 成员全局访问
  OR (users_collections.collection_uuid = ? 
      AND users_collections.read_only = false)  -- 直接授权且非只读
  OR groups.access_all = true  -- 组全局访问
  OR (collections_groups.collections_uuid IS NOT NULL 
      AND collections_groups.read_only = false)  -- 组关联且非只读
```

**合并规则总结**：
- `read_only`: 任一授权途径为 `false`，最终为 `false`（可写）
- `hide_passwords`: 任一授权途径为 `false`，最终为 `false`（可见密码）
- `manage`: 任一授权途径为 `true`，最终为 `true`（可管理）

### 4.3 特殊情况：Manager 角色的管理权限

在 `to_json_details` 中 [collection.rs: L107-L118](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L107-L118)，Manager 角色有额外的权限判定：

```rust
// Manager 类型用户，如果有读写权限（非 read_only 且非 hide_passwords）
// 则默认拥有 manage 权限
is_manager && (cu.manage || (!cu.read_only && !cu.hide_passwords))
```

这是一个**前端展示层面**的逻辑，用于兼容 Bitwarden 的权限模型。

---

## 五、组变更后的同步影响

### 5.1 数据版本更新机制 (Revision)

组变更会触发相关用户的 `revision_date` 更新，这是客户端同步的关键。

**核心函数**：[group.rs: L612-L617](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L612-L617) (`update_user_revision`)

```rust
pub async fn update_user_revision(&self, conn: &DbConn) {
    match Membership::find_by_uuid(&self.users_organizations_uuid, conn).await {
        Some(member) => User::update_uuid_revision(&member.user_uuid, conn).await,
        None => warn!("Member could not be found!"),
    }
}
```

### 5.2 触发同步的场景

#### 场景 1：组内成员变更

**位置**：[group.rs: L495-L496](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L495-L496) (`GroupUser::save`)

```rust
pub async fn save(&mut self, conn: &DbConn) -> EmptyResult {
    self.update_user_revision(conn).await;  // 入组/出组时更新
    // ... 保存逻辑
}
```

#### 场景 2：组-集合关联变更

**位置**：[group.rs: L321-L325](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L321-L325) (`CollectionGroup::save`)

```rust
pub async fn save(&mut self, org_uuid: &OrganizationId, conn: &DbConn) -> EmptyResult {
    let group_users = GroupUser::find_by_group(&self.groups_uuid, org_uuid, conn).await;
    for group_user in group_users {
        group_user.update_user_revision(conn).await;  // 组内所有成员都要更新
    }
    // ... 保存逻辑
}
```

**关键差异**：
- GroupUser 变更：只影响**单个用户**
- CollectionGroup 变更：影响**组内所有用户**

#### 场景 3：删除组

**位置**：[group.rs: L286-L288](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs#L286-L288) (`Group::delete`)

```rust
pub async fn delete(&self, org_uuid: &OrganizationId, conn: &DbConn) -> EmptyResult {
    CollectionGroup::delete_all_by_group(&self.uuid, org_uuid, conn).await?;
    GroupUser::delete_all_by_group(&self.uuid, org_uuid, conn).await?;
    // 删除组记录
}
```

删除组时，会级联删除：
1. 所有 `collections_groups` 关联 → 触发组内所有用户的 revision 更新
2. 所有 `groups_users` 关联 → 再次触发这些用户的 revision 更新

### 5.3 同步影响矩阵

| 操作 | 触发的 revision 更新 | 影响范围 |
|------|-------------------|---------|
| 添加用户到组 | 该用户 | 单个用户 |
| 从组移除用户 | 该用户 | 单个用户 |
| 组添加集合关联 | 组内所有用户 | 批量用户 |
| 组移除集合关联 | 组内所有用户 | 批量用户 |
| 修改组 access_all | 组内所有用户 | 批量用户 |
| 删除组 | 组内所有用户 (两次) | 批量用户 |
| 修改组名称 | 无 | - |

---

## 六、重要实现细节

### 6.1 前端展示的过滤逻辑

在 `to_json_user_details` 中 [organization.rs: L558-L605](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/organization.rs#L558-L605)，有一个重要的过滤：

```rust
// 如果用户通过组获得了 full_access，或者本身就是 access_all
// 则不返回单独的 collections 列表
let full_access_group = Group::is_in_full_access_group(...).await;
let collections: Vec<Value> = if include_collections && !(full_access_group || self.access_all) {
    // ... 返回直接授权的集合
    // 注意：通过组继承的集合不在此返回！
} else {
    Vec::with_capacity(0)
};
```

**重要**：通过组继承的集合权限，不会在用户详情接口中单独列出。客户端需要通过专门的组接口获取。

### 6.2 数据库 JOIN 链

查询用户通过组获得的集合权限时，JOIN 链非常长：

```
users_organizations (成员表)
  ← groups_users (组-成员关联)
    ← groups (组表)
    ← collections_groups (组-集合关联)
      ← collections (集合表)
```

代码示例 [collection.rs: L239-L251](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs#L239-L251)：

```rust
.left_join(groups_users::table.on(...))
.left_join(groups::table.on(...))
.left_join(collections_groups::table.on(...))
```

### 6.3 配置开关

所有组相关功能都受 `CONFIG.org_groups_enabled()` 开关控制，如果关闭则完全跳过组权限判定。

---

## 七、常见问题排查

### Q1: 用户加入组后看不到集合？
1. 检查组是否与该集合有关联 (`collections_groups` 表)
2. 检查用户的 `revision_date` 是否已更新
3. 检查用户是否已 Confirm 状态

### Q2: 修改组权限后客户端没刷新？
1. 确认相关用户的 `User.revision_date` 已更新
2. 客户端可能需要手动触发同步
3. 检查日志中是否有 "Failed to update revision" 警告

### Q3: access_all 组的成员为什么看不到某些集合？
`access_all = true` 只授予**访问权**，不影响集合本身的存在性。如果集合被删除或移动，仍然不可见。

### Q4: 直接授权和组授权的权限冲突怎么办？
遵循 **"最宽松原则"**：任一授权途径允许的权限，最终都会被允许。例如：
- 直接授权是 read_only=true，但组授权是 read_only=false → 最终可写
- 直接授权是 hide_passwords=false，但组授权是 hide_passwords=true → 最终密码可见

---

## 八、代码快速索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 组模型定义 | [group.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs) | L18-L30 |
| 组成员关联 | [group.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs) | L43-L49 |
| 访问权限判定 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs) | L149-L156 |
| 组全局权限判定 | [group.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs) | L593-L610 |
| 用户 revision 更新 | [group.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/group.rs) | L612-L617 |
| 新增/更新组 API | [organizations.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/api/core/organizations.rs) | L2611-L2650 |
| 删除组逻辑 | [organizations.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/api/core/organizations.rs) | L2688-L2717 |
| 获取用户可访问集合 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/18-vaultwarden/src/db/models/collection.rs) | L225-L301 |
