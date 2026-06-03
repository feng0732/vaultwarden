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
    └── Cipher (密码条目)
```

### 1.2 关键模型定义

#### Group (组)
- `src/db/models/group.rs` (L18-L30)
- 核心字段：
  - `access_all`: 组内成员是否可以访问组织内所有集合
  - `revision_date`: 组自身的版本更新时间

#### GroupUser (组-成员关联)
- `src/db/models/group.rs` (L43-L49)
- 关联 `groups_uuid` (组ID) 和 `users_organizations_uuid` (成员ID)
- 复合主键：(groups_uuid, users_organizations_uuid)

#### CollectionGroup (组-集合关联)
- `src/db/models/group.rs` (L32-L41)
- 权限字段：
  - `read_only`: 只读
  - `hide_passwords`: 隐藏密码
  - `manage`: 管理权限

#### CollectionUser (直接授权)
- `src/db/models/collection.rs` (L35-L44)
- 用户直接与集合建立关联，跳过组层级

---

## 二、成员关系：GroupUser 的绑定机制

### 2.1 关联表结构

`groups_users` 表的复合主键意味着：

- 一个成员可以加入多个组
- 一个组可以包含多个成员
- 但同一个成员在同一个组中只能有一条记录

### 2.2 成员入组流程

**API 入口**：`src/api/core/organizations.rs` (L2611-L2650) (`add_update_group` 函数)

```rust
// 新增/更新组时的成员绑定
for assigned_member in members {
    let mut user_entry = GroupUser::new(group.uuid.clone(), assigned_member.clone());
    user_entry.save(conn).await?;
    // 触发用户数据版本更新
}
```

### 2.3 更新组时的全量替换机制

**关键逻辑**：`src/api/core/organizations.rs` (L2594-L2595)

```rust
// 更新组时，先删除所有旧的关联，再重新添加
CollectionGroup::delete_all_by_group(&group_id, &org_id, &conn).await?;
GroupUser::delete_all_by_group(&group_id, &org_id, &conn).await?;
// 然后调用 add_update_group 重新添加
```

**重要提示**：这是一个 **全量替换** 操作，不是增量更新。前端需要传递完整的成员列表。

---

## 三、三层权限体系：可见性、展示、操作限制

### 3.1 第一层：Collection 可见性判定

**判定函数**：`src/db/models/collection.rs` (L149-L156) (`can_access_collection`)

```rust
pub async fn can_access_collection(member: &Membership, col_id: &CollectionId, conn: &DbConn) -> bool {
    member.has_status(MembershipStatus::Confirmed)
        && (member.has_full_access()           // 1. 成员全局权限
            || CollectionUser::has_access_to_collection_by_user(...)  // 2. 直接授权
            || (CONFIG.org_groups_enabled()
                && (GroupUser::has_full_access_by_member(...)         // 3. 组全局权限
                    || GroupUser::has_access_to_collection_by_member(...))))  // 4. 组集合关联
}
```

**规则**：
- **OR 逻辑**：四个条件任一满足即可访问
- 直接授权与组授权是**平等的 OR 关系**，没有优先级
- 只判定 "能不能看到/访问"，不判定具体权限细节

---

### 3.2 第二层：用户详情展示

**处理逻辑**：`src/db/models/organization.rs` (L556-L605) (`to_json_user_details`)

```rust
// 检查是否通过组获得 full_access
let full_access_group = Group::is_in_full_access_group(...).await;

// 如果有 full_access（成员或组），不返回单独的 collections 列表
let collections: Vec<Value> = if include_collections && !(full_access_group || self.access_all) {
    // 获取直接授权的集合
    let cu: HashMap<CollectionId, CollectionUser> = ...;
    // 获取组继承的集合（用于过滤）
    let cg: HashSet<CollectionId> = CollectionGroup::find_by_user(...).await;
    
    Collection::find_by_organization_and_user_uuid(...)
        .await
        .into_iter()
        .filter_map(|c| {
            if self.has_full_access() {
                // 成员全局权限
                Some(...)
            } else if let Some(cu) = cu.get(&c.uuid) {
                // 直接授权 - 显示
                Some(...)
            } else if cg.contains(&c.uuid) {
                // 组继承 - 不显示！跳过
                return None;
            } else {
                Some(...)
            }
        })
        .collect()
} else {
    Vec::with_capacity(0)
};
```

**规则**：
- **分离展示**：直接授权的集合在用户详情中列出，组继承的集合不列出
- 组继承的权限需要通过专门的组接口获取
- 如果成员或组有 `access_all = true`，则完全不返回 collections 字段

---

### 3.3 第三层：Cipher 操作权限限制（核心！）

**核心函数**：`src/db/models/cipher.rs` (L601-L669) (`get_access_restrictions`)

这是权限聚合最复杂也最重要的一层，决定了用户对具体密码条目的实际操作权限。

#### 3.3.1 快速通道：完全访问

```rust
// 如果满足任一条件，直接返回 (read_only=false, hide_passwords=false, manage=true)
if self.is_owned_by_user(user_uuid)                    // 1. 用户是 cipher 所有者
    || self.is_in_full_access_org(...)                  // 2. 成员全局权限
    || self.is_in_full_access_group(...)                // 3. 组全局权限
{
    return Some((false, false, true));
}
```

以上任一条件满足，用户拥有完全控制权，无需进一步判定。

#### 3.3.2 关键规则：直接授权覆盖组授权

**代码位置**：`src/db/models/cipher.rs` (L631-L638)

```rust
let rows = if let Some(cipher_sync_data) = cipher_sync_data {
    // ... 使用同步数据（见下一节）
} else {
    // 先查询直接授权
    let user_permissions = self.get_user_collections_access_flags(user_uuid, conn).await;
    
    // 重要！直接授权覆盖组授权！
    if user_permissions.is_empty() {
        // 只有当没有直接授权时，才考虑组授权
        self.get_group_collections_access_flags(user_uuid, conn).await
    } else {
        // 有直接授权，只使用直接授权，忽略组授权
        user_permissions
    }
};
```

**代码注释明确说明**：
```
// Also, user permission overrule group permissions
// and only user permissions are returned by the code above.
```

**结论**：**直接授权完全覆盖组授权**，不是取并集也不是聚合。如果用户对某个集合有直接授权，那么对该集合的组授权会被完全忽略。

#### 3.3.3 多集合权限聚合方式

当一个 cipher 属于多个集合时，需要聚合这些集合的权限：

**代码位置**：`src/db/models/cipher.rs` (L645-L666)

```rust
// For a flag to be in effect for a cipher, upstream
// requires all collections the cipher is in to have that flag set.
// Therefore, we do a boolean AND of all values in each of the `read_only`
// and `hide_passwords` columns.
//
// The only exception is for the `manage` flag, that needs a boolean OR!

let mut read_only = true;
let mut hide_passwords = true;
let mut manage = false;
for (ro, hp, mn) in &rows {
    read_only &= ro;        // AND：所有集合都只读，才是只读
    hide_passwords &= hp;   // AND：所有集合都隐藏密码，才隐藏密码
    manage |= mn;           // OR：任一集合可管理，就可管理
}
```

**聚合规则总结**：

| 权限字段 | 聚合方式 | 说明 |
|---------|---------|------|
| `read_only` | **AND** | 所有集合都设为只读，最终才是只读<br>任一集合可写，最终就可写 |
| `hide_passwords` | **AND** | 所有集合都设为隐藏密码，最终才隐藏<br>任一集合可见密码，最终就可见 |
| `manage` | **OR** | 任一集合设为可管理，最终就可管理<br>所有集合都不可管理，最终才不可管理 |

---

### 3.4 CipherSyncData 中的组内权限聚合

当使用 CipherSyncData 进行批量同步时，还有一层组内权限的预聚合：

**代码位置**：`src/api/core/ciphers.rs` (L2175-L2190)

```rust
// 同一用户对同一集合可能通过多个组获得访问权限
// 需要先把这些组的权限聚合起来
let user_collections_groups: HashMap<CollectionId, CollectionGroup> = 
    CollectionGroup::find_by_user(user_id, conn).await.into_iter().fold(
        HashMap::new(),
        |mut combined_permissions, cg| {
            combined_permissions
                .entry(cg.collections_uuid.clone())
                .and_modify(|existing| {
                    // 同一集合的多个组权限，取最宽松的设置
                    existing.read_only &= cg.read_only;        // AND：任一组可写就可写
                    existing.hide_passwords &= cg.hide_passwords;  // AND：任一组可见就可见
                    existing.manage |= cg.manage;             // OR：任一组可管理就可管理
                })
                .or_insert(cg);
            combined_permissions
        },
    );
```

**组内聚合规则**（对同一集合的多个组授权）：
- `read_only`: AND → 任一组可写，最终可写
- `hide_passwords`: AND → 任一组可见密码，最终可见
- `manage`: OR → 任一组可管理，最终可管理

这与多集合聚合的规则一致，都是**取最宽松**的权限。

---

### 3.5 三层权限体系对比总结

| 层面 | 判定逻辑 | 直接授权 vs 组授权 | 聚合方式 |
|------|---------|-------------------|---------|
| **Collection 可见性** | OR 逻辑，任一途径即可访问 | 平等 OR 关系 | 不涉及 |
| **用户详情展示** | 分离展示 | 分开显示，组继承的不列出 | 不涉及 |
| **Cipher 操作限制** | 分层次判定 | **直接授权完全覆盖组授权** | read_only: AND<br>hide_passwords: AND<br>manage: OR |

---

## 四、权限冲突场景分析

### 场景 1：直接授权 vs 组授权（同一集合）

**情况**：
- 用户 A 对集合 X 有直接授权：`read_only=true, hide_passwords=true, manage=false`
- 用户 A 同时通过组 G 对集合 X 有授权：`read_only=false, hide_passwords=false, manage=true`

**结果**：
- **直接授权完全覆盖组授权**
- 实际权限：`read_only=true, hide_passwords=true, manage=false`
- 组授权的宽松权限被完全忽略

---

### 场景 2：同一 cipher 跨多个集合（均为直接授权）

**情况**：
- Cipher C 同时属于集合 X 和集合 Y
- 用户对集合 X 的直接授权：`read_only=true, hide_passwords=true, manage=false`
- 用户对集合 Y 的直接授权：`read_only=false, hide_passwords=false, manage=true`

**结果**：
- `read_only = true AND false = false` → 可写
- `hide_passwords = true AND false = false` → 可见密码
- `manage = false OR true = true` → 可管理
- 最终权限最宽松：`(false, false, true)`

---

### 场景 3：同一 cipher 跨多个集合（混合授权）

**情况**：
- Cipher C 同时属于集合 X 和集合 Y
- 用户对集合 X 有直接授权：`read_only=true, hide_passwords=true, manage=false`
- 用户对集合 Y 只有组授权：`read_only=false, hide_passwords=false, manage=true`

**结果**：
- 集合 X 使用直接授权
- 集合 Y：因为用户对 X 有直接授权，所以对 Y 的组授权**完全被忽略**
- 最终只聚合直接授权：`(true, true, false)`
- 注意：这可能导致比预期更严格的权限！

---

### 场景 4：同一集合通过多个组授权

**情况**：
- 用户 A 对集合 X 没有直接授权
- 组 G1 对集合 X：`read_only=true, hide_passwords=true, manage=false`
- 组 G2 对集合 X：`read_only=false, hide_passwords=true, manage=true`
- 用户 A 同时属于 G1 和 G2

**结果**（CipherSyncData 预聚合）：
- `read_only = true AND false = false` → 可写
- `hide_passwords = true AND true = true` → 隐藏密码
- `manage = false OR true = true` → 可管理
- 聚合后：`(false, true, true)`

---

## 五、组变更后的同步影响：Revision 更新机制

### 5.1 同步触发的核心原理

**核心函数**：`src/db/models/group.rs` (L612-L617) (`GroupUser::update_user_revision`)

```rust
pub async fn update_user_revision(&self, conn: &DbConn) {
    match Membership::find_by_uuid(&self.users_organizations_uuid, conn).await {
        Some(member) => User::update_uuid_revision(&member.user_uuid, conn).await,
        None => warn!("Member could not be found!"),
    }
}
```

该函数通过 GroupUser 记录找到对应的成员，再找到成员关联的用户，然后更新 `User` 表的 `revision_date`。客户端通过比较 revision_date 判断是否需要同步数据。

---

### 5.2 场景一：组基本信息变更（名称、access_all）

**关键发现**：`Group::save()` 本身**不会**触发用户的 revision 更新，它只更新组自身的 `revision_date`。

但是，**在 API 层的 `put_group` 实现中** (`src/api/core/organizations.rs` L2570-L2608)，采用了**全量替换策略**：

```rust
async fn put_group(...) {
    // 1. 更新组基本信息（名称、access_all）
    let updated_group = group_request.update_group(group);
    
    // 2. 先删除所有旧关联 -> 这里会触发用户 revision 更新
    CollectionGroup::delete_all_by_group(&group_id, &org_id, &conn).await?;  // 触发组内所有用户
    GroupUser::delete_all_by_group(&group_id, &org_id, &conn).await?;        // 再次触发组内所有用户
    
    // 3. 重新添加所有关联 -> 这里也会触发用户 revision 更新
    add_update_group(updated_group, ...).await
}
```

**实际影响**：
- 即使只修改**组名称**，由于删除+重新添加关联的副作用，组内**所有用户**的 revision 都会被更新
- 修改 `access_all` 同样会触发组内所有用户的 revision 更新

**代码路径**：
- 删除 CollectionGroup 触发：`src/db/models/group.rs` (L456-L469)
- 删除 GroupUser 触发：`src/db/models/group.rs` (L639-L652)

---

### 5.3 场景二：组成员变更（添加/移除）

#### 5.3.1 添加成员到组

**触发点**：`src/db/models/group.rs` (L494-L496) (`GroupUser::save`)

```rust
pub async fn save(&mut self, conn: &DbConn) -> EmptyResult {
    self.update_user_revision(conn).await;  // 保存前先更新用户 revision
    // ... 保存到数据库
}
```

**影响范围**：仅被添加的**单个用户**

#### 5.3.2 从组移除成员

**触发点 1 - 单个移除**：`src/db/models/group.rs` (L619-L637) (`GroupUser::delete_by_group_and_member`)

```rust
pub async fn delete_by_group_and_member(...) -> EmptyResult {
    // 删除前先更新用户 revision
    match Membership::find_by_uuid(member_uuid, conn).await {
        Some(member) => User::update_uuid_revision(&member.user_uuid, conn).await,
        None => warn!("Member could not be found!"),
    }
    // ... 执行删除
}
```

**触发点 2 - 全组清空**：`src/db/models/group.rs` (L639-L652) (`GroupUser::delete_all_by_group`)

```rust
pub async fn delete_all_by_group(group_uuid, org_uuid, conn) -> EmptyResult {
    let group_users = GroupUser::find_by_group(group_uuid, org_uuid, conn).await;
    for group_user in group_users {
        group_user.update_user_revision(conn).await;  // 逐个更新组内所有用户
    }
    // ... 执行批量删除
}
```

**影响范围**：
- 单个移除：仅被移除的**单个用户**
- 全组清空：组内**所有用户**

---

### 5.4 场景三：组-集合关联变更（添加/移除/修改权限）

#### 5.4.1 添加/修改集合关联

**触发点**：`src/db/models/group.rs` (L321-L325) (`CollectionGroup::save`)

```rust
pub async fn save(&mut self, org_uuid: &OrganizationId, conn: &DbConn) -> EmptyResult {
    let group_users = GroupUser::find_by_group(&self.groups_uuid, org_uuid, conn).await;
    for group_user in group_users {
        group_user.update_user_revision(conn).await;  // 组内所有成员都要更新
    }
    // ... 保存到数据库
}
```

#### 5.4.2 移除集合关联

**触发点 - 单个移除**：`src/db/models/group.rs` (L440-L454) (`CollectionGroup::delete`)

```rust
pub async fn delete(&self, org_uuid: &OrganizationId, conn: &DbConn) -> EmptyResult {
    let group_users = GroupUser::find_by_group(&self.groups_uuid, org_uuid, conn).await;
    for group_user in group_users {
        group_user.update_user_revision(conn).await;  // 组内所有成员都要更新
    }
    // ... 执行删除
}
```

**触发点 - 组的所有集合关联清空**：`src/db/models/group.rs` (L456-L469) (`CollectionGroup::delete_all_by_group`)

```rust
pub async fn delete_all_by_group(group_uuid, org_uuid, conn) -> EmptyResult {
    let group_users = GroupUser::find_by_group(group_uuid, org_uuid, conn).await;
    for group_user in group_users {
        group_user.update_user_revision(conn).await;  // 组内所有成员都要更新
    }
    // ... 执行批量删除
}
```

**影响范围**：组内**所有用户**（无论组有多少成员，每个成员的 revision 都会更新）

---

### 5.5 场景四：删除整个组

**触发点**：`src/db/models/group.rs` (L286-L296) (`Group::delete`)

```rust
pub async fn delete(&self, org_uuid: &OrganizationId, conn: &DbConn) -> EmptyResult {
    // 1. 删除所有集合关联 -> 触发组内所有用户 revision 更新
    CollectionGroup::delete_all_by_group(&self.uuid, org_uuid, conn).await?;
    
    // 2. 删除所有成员关联 -> 再次触发组内所有用户 revision 更新
    GroupUser::delete_all_by_group(&self.uuid, org_uuid, conn).await?;
    
    // 3. 删除组本身
    // ...
}
```

**影响范围**：
- 组内**所有用户**会被触发**两次** revision 更新
- 第一次：删除 CollectionGroup 时触发 (`delete_all_by_group`)
- 第二次：删除 GroupUser 时触发 (`delete_all_by_group`)

---

### 5.6 同步影响矩阵

| 变更操作 | 触发的 Revision 更新 | 影响范围 | 代码位置 |
|---------|-------------------|---------|---------|
| **修改组名称** | 是（间接，通过全量替换） | 组内所有用户 | `src/api/core/organizations.rs` (L2594-L2595) |
| **修改组 access_all** | 是（间接，通过全量替换） | 组内所有用户 | `src/api/core/organizations.rs` (L2594-L2595) |
| **添加成员到组** | 是（直接） | 单个用户 | `src/db/models/group.rs` (L495-L496) |
| **从组移除成员** | 是（直接） | 单个用户 | `src/db/models/group.rs` (L624-L626) |
| **组添加集合关联** | 是（直接） | 组内所有用户 | `src/db/models/group.rs` (L322-L325) |
| **组移除集合关联** | 是（直接） | 组内所有用户 | `src/db/models/group.rs` (L441-L444) |
| **修改组-集合权限** | 是（直接） | 组内所有用户 | `src/db/models/group.rs` (L322-L325) |
| **删除整个组** | 是（直接，两次） | 组内所有用户 | `src/db/models/group.rs` (L287-L288) |
| **仅调用 Group::save()** | 否 | - | `src/db/models/group.rs` (L165-L196) |

---

### 5.7 注意事项

1. **间接触发**：单纯修改组名称/access_all 本身不会触发用户更新，但由于 `put_group` API 采用"先删后加"的全量替换策略，实际上会触发所有用户更新。

2. **重复更新**：在某些场景下（如删除组），同一用户的 revision 可能会被更新多次，这是正常的设计。

3. **性能考量**：对于大组（成员众多），修改集合关联或删除组可能导致大量数据库写操作。

4. **客户端同步**：用户 revision 更新后，客户端下次拉取时会检测到变化并重新同步相关数据。

---

## 六、重要实现细节

### 6.1 数据库 JOIN 链

查询用户通过组获得的集合权限时，JOIN 链非常长：

```
users_organizations (成员表)
  ← groups_users (组-成员关联)
    ← groups (组表)
    ← collections_groups (组-集合关联)
      ← collections (集合表)
        ← ciphers_collections (集合-密码关联)
          ← ciphers (密码表)
```

### 6.2 配置开关

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
**直接授权完全覆盖组授权**！如果用户对某个集合有直接授权，组授权会被完全忽略。这不是取并集，而是直接使用直接授权的权限。

### Q5: 为什么修改组名称后所有用户都需要重新同步？
因为 `put_group` API 采用全量替换策略，先删除所有成员和集合关联再重新添加，这个过程会间接触发所有组内用户的 revision 更新。

### Q6: 一个 cipher 属于多个集合时，权限怎么算？
- `read_only` 和 `hide_passwords` 用 **AND** 聚合：所有集合都限制，才最终限制
- `manage` 用 **OR** 聚合：任一集合允许管理，就最终可管理

### Q7: 用户通过多个组访问同一集合，权限怎么算？
在 CipherSyncData 预聚合阶段：
- `read_only`: AND → 任一组可写，最终可写
- `hide_passwords`: AND → 任一组可见密码，最终可见
- `manage`: OR → 任一组可管理，最终可管理
- 总体原则：**取最宽松**的权限

---

## 八、代码快速索引

| 功能 | 文件路径 | 行号范围 |
|------|---------|---------|
| 组模型定义 | `src/db/models/group.rs` | L18-L30 |
| 组成员关联模型 | `src/db/models/group.rs` | L43-L49 |
| 组-集合关联模型 | `src/db/models/group.rs` | L32-L41 |
| Collection 可见性判定 | `src/db/models/collection.rs` | L149-L156 |
| 用户详情展示（集合过滤） | `src/db/models/organization.rs` | L556-L605 |
| Cipher 权限限制核心函数 | `src/db/models/cipher.rs` | L601-L669 |
| 直接授权覆盖组授权逻辑 | `src/db/models/cipher.rs` | L631-L638 |
| 多集合权限聚合规则 | `src/db/models/cipher.rs` | L645-L666 |
| CipherSyncData 组权限预聚合 | `src/api/core/ciphers.rs` | L2175-L2190 |
| 组全局权限判定 | `src/db/models/group.rs` | L593-L610 |
| 组集合权限判定 | `src/db/models/group.rs` | L569-L591 |
| 用户 revision 更新函数 | `src/db/models/group.rs` | L612-L617 |
| GroupUser 保存（添加成员）| `src/db/models/group.rs` | L494-L539 |
| GroupUser 批量删除 | `src/db/models/group.rs` | L639-L652 |
| CollectionGroup 保存 | `src/db/models/group.rs` | L321-L378 |
| CollectionGroup 批量删除 | `src/db/models/group.rs` | L456-L469 |
| 组删除（级联触发） | `src/db/models/group.rs` | L286-L296 |
| 新增/更新组 API | `src/api/core/organizations.rs` | L2611-L2650 |
| 更新组 API（全量替换）| `src/api/core/organizations.rs` | L2570-L2609 |
| 删除组 API 实现 | `src/api/core/organizations.rs` | L2688-L2717 |
| 获取用户可访问集合 | `src/db/models/collection.rs` | L225-L301 |
| 可写权限判定 | `src/db/models/collection.rs` | L426-L504 |
