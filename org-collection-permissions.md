# Vaultwarden 组织与集合权限叠加分析

## 一、成员角色体系

### 1.1 角色类型定义

在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L95-L101) 中定义了四种成员角色：

| 角色 | 值 | 说明 |
|------|-----|------|
| Owner (所有者) | 0 | 最高权限，可管理所有设置 |
| Admin (管理员) | 1 | 仅次于所有者，可管理用户和集合 |
| User (普通用户) | 2 | 基础用户，只能访问分配的集合 |
| Manager (管理者) | 3 | 可管理指定集合 |

### 1.2 角色权限层级

在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L121-L138) 中定义了角色的比较逻辑：

```rust
const ACCESS_LEVEL: [i32; 4] = [
    3, // Owner (最高)
    2, // Admin
    0, // User (最低)
    1, // Manager && Custom
];
```

权限层级从高到低：**Owner > Admin > Manager > User**

### 1.3 Custom 角色的特殊处理

Vaultwarden 将 Custom (自定义) 角色映射为 Manager 角色：

```rust
// 在 MembershipType::from_str() 中
"4" | "Custom" => Some(MembershipType::Manager),
```

这是一个 HACK，在 [send_invite()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1042-L1063) 和 [edit_member()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1535-L1550) 中都有体现。

---

## 二、access_all 标志的作用

### 2.1 定义

在 [Membership 结构体](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L54) 中：

```rust
pub access_all: bool,
```

### 2.2 has_full_access() 判断逻辑

在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L818-L820) 中：

```rust
pub fn has_full_access(&self) -> bool {
    (self.access_all || self.atype >= MembershipType::Admin) && self.has_status(MembershipStatus::Confirmed)
}
```

**关键点：**
- Admin 及以上角色 **自动拥有** full_access
- Manager 及以下角色需要 `access_all = true` 才拥有 full_access
- 必须是 Confirmed 状态才生效

### 2.3 Custom 角色的 access_all 判定

当 Custom 角色勾选了"管理所有集合"时，通过三个权限标志联合判定：

```rust
// 在 send_invite() 和 edit_member() 中
let access_all = new_type >= MembershipType::Admin
    || (raw_type.eq("4")  // Custom 角色
        && data.permissions.get("editAnyCollection") == Some(&json!(true))
        && data.permissions.get("deleteAnyCollection") == Some(&json!(true))
        && data.permissions.get("createNewCollections") == Some(&json!(true)));
```

---

## 三、集合访问权限叠加机制

### 3.1 权限授予的三个来源

用户对集合的访问权限来自三个来源，在 [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L149-L156) 的 `can_access_collection()` 中定义：

```rust
pub async fn can_access_collection(member: &Membership, col_id: &CollectionId, conn: &DbConn) -> bool {
    member.has_status(MembershipStatus::Confirmed)
        && (member.has_full_access()  // 1. 组织级别的 full_access
            || CollectionUser::has_access_to_collection_by_user(col_id, &member.user_uuid, conn).await  // 2. 直接分配给用户
            || (CONFIG.org_groups_enabled()
                && (GroupUser::has_full_access_by_member(&member.org_uuid, &member.uuid, conn).await  // 3a. 组级别的 full_access
                    || GroupUser::has_access_to_collection_by_member(col_id, &member.uuid, conn).await)))  // 3b. 通过组分配
}
```

### 3.2 集合权限的三个维度

每个集合分配有三个权限标志，在 [CollectionUser](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L38-L44) 中定义：

| 标志 | 类型 | 说明 |
|------|------|------|
| read_only | bool | 是否只读（true = 不能编辑） |
| hide_passwords | bool | 是否隐藏密码（true = 不能查看密码） |
| manage | bool | 是否可管理（true = 可管理集合成员） |

### 3.3 权限叠加的 OR 逻辑

用户只要通过 **任一来源** 获得访问权限，即可访问集合：

```
用户能访问集合 = 
  (用户是 Admin/Owner) OR
  (用户 access_all = true) OR
  (用户被直接分配到该集合) OR
  (用户所在组的 access_all = true) OR
  (用户所在组被分配到该集合)
```

---

## 四、共享条目（Cipher）可见性计算

### 4.1 访问限制计算入口

在 [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/cipher.rs#L601-L669) 的 `get_access_restrictions()` 方法中：

```rust
pub async fn get_access_restrictions(
    &self,
    user_uuid: &UserId,
    cipher_sync_data: Option<&CipherSyncData>,
    conn: &DbConn,
) -> Option<(bool, bool, bool)> {
    // 返回: Some((read_only, hide_passwords, manage)) 或 None（无访问权限）
}
```

### 4.2 第一级：完全访问检查（短路返回）

如果满足以下任一条件，直接返回无限制访问 `(false, false, true)`：

```rust
if self.is_owned_by_user(user_uuid)                    // 1. 用户直接拥有
    || self.is_in_full_access_org(...)                 // 2. 组织级 full_access
    || self.is_in_full_access_group(...)               // 3. 组级 full_access
{
    return Some((false, false, true));
}
```

### 4.3 第二级：集合权限收集

如果没有完全访问，则收集用户对该条目所在所有集合的权限：

```rust
// 优先使用用户直接分配的权限
let user_permissions = self.get_user_collections_access_flags(...).await;
if user_permissions.is_empty() {
    // 如果没有直接分配，使用组分配的权限
    user_permissions = self.get_group_collections_access_flags(...).await;
}
```

**重要：用户直接权限优先于组权限**

### 4.4 第三级：多集合权限合并

当条目存在于多个集合中时，权限合并规则：

| 标志 | 合并逻辑 | 说明 |
|------|----------|------|
| read_only | AND | **所有** 集合都只读时，条目才只读 |
| hide_passwords | AND | **所有** 集合都隐藏密码时，条目才隐藏密码 |
| manage | OR | **任一** 集合可管理时，条目就可管理 |

```rust
let mut read_only = true;
let mut hide_passwords = true;
let mut manage = false;
for (ro, hp, mn) in &rows {
    read_only &= ro;           // 只要有一个集合可写 → 整体可写
    hide_passwords &= hp;      // 只要有一个集合显示密码 → 整体显示密码
    manage |= mn;              // 只要有一个集合可管理 → 整体可管理
}
```

### 4.5 权限判定流程图

```
用户访问条目
    ↓
是条目所有者? ──是──→ 无限制访问 (r/w, 可见密码, 可管理)
    ↓否
有组织级 full_access? ──是──→ 无限制访问
    ↓否
有组级 full_access? ──是──→ 无限制访问
    ↓否
获取用户直接分配的集合权限列表
    ↓
列表为空? ──是──→ 获取组分配的集合权限列表
    ↓否/是
权限列表为空? ──是──→ 无访问权限 (None)
    ↓否
对所有集合权限做合并:
  read_only     = 所有集合的 read_only 做 AND
  hide_passwords = 所有集合的 hide_passwords 做 AND
  manage        = 所有集合的 manage 做 OR
    ↓
返回 (read_only, hide_passwords, manage)
```

---

## 五、移除成员后的访问边界清理

### 5.1 删除成员时的清理流程

在 [organizations.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1677-L1719) 的 `delete_member_impl()` 中：

```rust
async fn delete_member_impl(...) -> EmptyResult {
    // 1. 权限检查
    // 2. 记录事件日志
    // 3. 发送用户更新通知
    // 4. 调用 Membership::delete()
    member_to_delete.delete(conn).await
}
```

### 5.2 Membership::delete() 的级联清理

在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L772-L784) 中：

```rust
pub async fn delete(self, conn: &DbConn) -> EmptyResult {
    User::update_uuid_revision(&self.user_uuid, conn).await;

    // 清理用户-集合关联
    CollectionUser::delete_all_by_user_and_org(&self.user_uuid, &self.org_uuid, conn).await?;
    
    // 清理用户-组关联
    GroupUser::delete_all_by_member(&self.uuid, conn).await?;

    // 删除成员记录
    conn.run(move |conn| {
        diesel::delete(users_organizations::table.filter(...))
            .execute(conn)
    }).await
}
```

### 5.3 清理的边界范围

| 清理对象 | 说明 | 相关代码 |
|----------|------|----------|
| 用户-集合关联 | 删除该用户在该组织内所有的集合分配 | [CollectionUser::delete_all_by_user_and_org()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L830-L852) |
| 用户-组关联 | 删除该用户在该组织内所有的组成员关系 | [GroupUser::delete_all_by_member()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/group.rs#L654-L667) |
| 用户安全戳更新 | 触发客户端重新同步 | [User::update_uuid_revision()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/user.rs) |

### 5.4 撤销（Revoke） vs 删除（Delete）

**撤销成员** 在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L281-L287) 中：

```rust
pub fn revoke(&mut self) -> bool {
    if self.status > MembershipStatus::Revoked as i32 {
        self.status -= ACTIVATE_REVOKE_DIFF;  // 减去 128
        return true;
    }
    false
}
```

**撤销 vs 删除的区别：**

| 操作 | 成员记录 | 集合分配 | 组成员 | 可恢复 |
|------|----------|----------|--------|--------|
| Revoke (撤销) | 保留，status 减 128 | 保留 | 保留 | 是（restore） |
| Delete (删除) | 彻底删除 | 级联删除 | 级联删除 | 否 |

### 5.5 撤销后的权限检查

在 [auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/auth.rs#L703-L707) 的 `OrgHeaders::is_member()` 中：

```rust
fn is_member(&self) -> bool {
    // 已撤销用户（status < 0）无法通过检查
    self.membership_status != MembershipStatus::Revoked && self.membership_type >= MembershipType::User
}
```

**注意：** 撤销后用户的集合分配和组成员关系仍然保留，只是状态变为 Revoked。恢复后权限立即恢复生效。

---

## 六、权限检查的 Request Guard 机制

### 6.1 权限层级 Guard

在 [auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/auth.rs) 中定义了多层权限 Guard：

| Guard | 要求 | 用途 |
|-------|------|------|
| Headers | 已登录用户 | 个人金库操作 |
| OrgMemberHeaders | 组织成员（非撤销） | 组织基本操作 |
| ManagerHeadersLoose | Manager+ 且 Confirmed | 集合列表等 |
| ManagerHeaders | Manager+ 且 Confirmed + 集合管理权限 | 特定集合操作 |
| AdminHeaders | Admin+ 且 Confirmed | 用户管理等 |
| OwnerHeaders | Owner 且 Confirmed | 组织删除等 |

### 6.2 ManagerHeaders 的集合权限检查

在 [auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/auth.rs#L861-L890) 中：

```rust
async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error> {
    let headers = try_outcome!(OrgHeaders::from_request(request).await);
    if headers.is_confirmed_and_manager() {
        if let Some(col_id) = get_col_id(request) {
            // 验证用户对该集合有管理权限
            if !Collection::is_coll_manageable_by_user(&col_id, &headers.membership.user_uuid, &conn).await {
                err_handler!("The current user isn't a manager for this collection")
            }
        }
        // ...
    }
}
```

---

## 七、关键代码位置速查表

| 功能 | 文件 | 关键方法 |
|------|------|----------|
| 角色定义 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs) | `MembershipType` enum |
| full_access 判断 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L818) | `has_full_access()` |
| 集合可访问性 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L149) | `can_access_collection()` |
| 条目权限计算 | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/cipher.rs#L601) | `get_access_restrictions()` |
| 删除成员清理 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L772) | `Membership::delete()` |
| 成员撤销/恢复 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L273) | `revoke()`, `restore()` |
| 权限 Guard | [auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/auth.rs) | `OrgHeaders`, `AdminHeaders` 等 |
