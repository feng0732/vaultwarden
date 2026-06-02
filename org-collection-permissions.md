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

### 2.3 access_all 实际会落到哪些角色？

**关键代码事实**：

在 [send_invite()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1059-L1063) 和 [edit_member()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1546-L1550) 中：

```rust
let access_all = new_type >= MembershipType::Admin
    || (raw_type.eq("4")  // Custom/Manager 角色
        && data.permissions.get("editAnyCollection") == Some(&json!(true))
        && ...);
```

**实际能获得 access_all=true 的只有三类角色：**

| 角色 | access_all 设置方式 | has_full_access() 结果 |
|--------|-------------------|---------------------|
| Owner | 自动设置为 true | true（因为 atype >= Admin） |
| Admin | 自动设置为 true | true（因为 atype >= Admin） |
| Manager(Custom) | 勾选"管理所有集合"时设为 true | access_all=true 时为 true |
| User | **永远不会被设置为 true** | 永远为 false |

**User 角色不可能获得组织级 full_access。代码注释也明确说明：[collection.rs L102](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L102)

```rust
// Users are not able to have full access
```

---

## 三、集合访问权限叠加机制

### 3.1 权限授予的三个来源

用户对集合的访问权限来自三个来源，在 [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L149-L156) 的 `can_access_collection()` 中定义：

```rust
pub async fn can_access_collection(member: &Membership, col_id: &CollectionId, conn: &DbConn) -> bool {
    member.has_status(MembershipStatus::Confirmed)
        && (member.has_full_access()  // 1. 组织级别的 full_access（仅 Owner/Admin/Manager+access_all
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
  (用户是 Manager 且 access_all = true) OR
  (用户被直接分配到该集合) OR
  (用户所在组的 access_all = true) OR
  (用户所在组被分配到该集合)
```

---

## 四、集合详情中 readOnly / hidePasswords / manage 的完整推导

这是理解权限叠加最关键的部分。三个标志的最终值由**组织角色 + access_all + 用户直接分配 + 组分配**四层因素联合决定，且在不同的序列化场景下有不同的计算路径。

### 4.1 场景一：同步时 Collection.to_json_details() — 面向当前用户自己

在 [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L93-L147) 中，当客户端执行 `/sync` 时，每个集合以 `collectionDetails` 对象返回给用户：

```rust
let (read_only, hide_passwords, manage) =
    // 路径 A: 组织级 full_access 用户（Owner/Admin/Manager+access_all）
    if m.has_full_access() => (false, false, m.atype >= MembershipType::Manager)

    // 路径 B: 有用户直接分配
    else if let Some(cu) = cipher_sync_data.user_collections.get(&self.uuid) => (
        cu.read_only,
        cu.hide_passwords,
        is_manager && (cu.manage || (!cu.read_only && !cu.hide_passwords)),
    )

    // 路径 C: 只有组分配
    else if let Some(cg) = cipher_sync_data.user_collections_groups.get(&self.uuid) => (
        cg.read_only,
        cg.hide_passwords,
        is_manager && (cg.manage || (!cg.read_only && !cg.hide_passwords)),
    )

    // 路径 D: 都没有（理论上不应到达）
    else => (false, false, false)
;
```

**逐路径分析：

#### 路径 A：组织级 full_access

当 `has_full_access()` 返回 true 时：
- `readOnly = false`，`hidePasswords = false`：Admin/Owner 或 Manager+access_all 用户对**所有集合**拥有完整读写、密码可见权限，集合级限制完全无效
- `manage = m.atype >= MembershipType::Manager`：
  - Owner/Admin → manage=true（因为 >= Manager）
  - Manager+access_all → manage=true
  - **注意：User 角色永远走不到这条路径

#### 路径 B：用户直接分配（read_only / hide_passwords 取原始值，manage 受角色提升）

- `readOnly` 和 `hidePasswords` 直接取 `CollectionUser` 中存储的原始值
- `manage` 的计算最复杂：
  ```
  manage = is_manager && (cu.manage || (!cu.read_only && !cu.hide_passwords))
  ```
  拆解：**只有 Manager 角色才能获得 manage=true**，且需要满足以下任一条件：
  - 集合明确标记了 `cu.manage = true`，或者
  - 用户对该集合同时拥有写权限（`!read_only`）和密码可见权限（`!hide_passwords`）

  也就是说，一个普通 User 即使在 `CollectionUser` 中被标记 `manage = true`，在同步时也会被降为 `manage = false`。**角色是 manage 的前置门槛**。

#### 路径 C：组分配（逻辑与路径 B 相同，但取 CollectionGroup 的值）

与路径 B 完全对称，只是数据来源从 `user_collections` 换成了 `user_collections_groups`。

#### 关键发现：组级 full_access 在这条路径里**完全没有被检查！

`to_json_details()` 只检查 `m.has_full_access()`（组织级），但**没有检查用户是否属于 access_all=true 的组**。这意味着：

- **组级 full_access 的用户在集合详情中不会触发 full_access 路径**，而是会走到路径 C（组分配）。

- 因为组级 full_access 的用户，其 CollectionGroup 记录在 CipherSyncData 构建时被预合并了所有集合的权限，所以仍能看到所有集合，但权限值是从组分配来的，不是从 full_access 路径来的。

#### 路径 D 的含义

如果走到这里，说明用户既没有直接分配也没有组分配。在 CipherSyncData 预构建逻辑中，这通常不会发生（因为 `find_by_user_uuid` 只返回用户有权限的集合），但作为防御返回 `(false, false, false)`。

### 4.2 场景二：管理员查看成员详情 — Membership.to_json_user_details()

在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L535-L608) 中，管理员查看组织成员列表时，每个成员的 `collections` 字段展示该成员被分配的集合及权限：

```rust
// 先判断是否在 full_access 组中
let full_access_group = CONFIG.org_groups_enabled()
    && Group::is_in_full_access_group(&self.user_uuid, &self.org_uuid, conn).await;

// 如果 full_access_group 或 access_all 为 true，collections 返回空数组
let collections: Vec<Value> = if include_collections && !(full_access_group || self.access_all) {
    // ... 返回单独的集合列表
} else {
    Vec::with_capacity(0)  // 空数组
};
```

#### 为什么 full_access 时不返回单独 collection？

原因有三层：

1. **语义冗余**：`access_all = true`（或组 `access_all = true`）意味着用户已经能访问**所有**集合，列出单独的集合没有信息增量
2. **Bitwarden 客户端约定**：客户端看到 `accessAll: true` + `collections: []` 时，理解为"全部集合可访问"；如果同时返回了 `collections` 列表，客户端可能会混淆
3. **组分配不在此处返回**：代码中 [第592-593行](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L590-L596) 明确注释了——组分配的集合由**专门的组端点**返回，不混入成员详情的 collections 数组：

```rust
} else if cg.contains(&c.uuid) {
    // If previous checks failed it might be that this user has access via a group,
    // but we should not return those elements here
    // Those are returned via a special group endpoint
    return None;
}
```

#### 非全访问用户集合权限的三路分支

当 `!(full_access_group || self.access_all)` 时，进入集合列表构建：

```rust
let (read_only, hide_passwords, manage) =
    // 分支 1: has_full_access()（角色为 Admin/Owner 且 Confirmed）
    if self.has_full_access() => (false, false, self.atype >= MembershipType::Manager)

    // 分支 2: 有用户直接分配
    else if let Some(cu) = cu.get(&c.uuid) => (
        cu.read_only,
        cu.hide_passwords,
        cu.manage || (self.atype == MembershipType::Manager && !cu.read_only && !cu.hide_passwords),
    )

    // 分支 3: 只有组分配 → 跳过，返回 None
    else if cg.contains(&c.uuid) => return None

    // 分支 4: 无分配（异常防御）
    else => (true, true, false)
;
```

**关键区别**：这里的分支 2 中 manage 的计算与 `to_json_details()` 不同：

| 场景 | manage 公式 | 区别 |
|------|------------|------|
| `to_json_details()`（自己看） | `is_manager && (cu.manage || (!ro && !hp))` | Manager 角色是前置条件 |
| `to_json_user_details()`（管理员看成员） | `cu.manage || (is_manager && !ro && !hp))` | cu.manage 直接生效，角色只是额外提升 |

在管理员视角，`cu.manage = true` 时不论角色都显示 manage=true，因为这反映的是**实际的集合级配置**；而在用户同步视角，只有 Manager 角色才能行使管理权。

### 4.3 场景三：集合详情页查看成员 — CollectionMembership.to_json_details_for_member()

在 [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L941-L952) 中，查看某个集合的成员列表时：

```rust
pub fn to_json_details_for_member(&self, membership_type: i32) -> Value {
    json!({
        "id": self.membership_uuid,
        "readOnly": self.read_only,
        "hidePasswords": self.hide_passwords,
        "manage": membership_type >= MembershipType::Admin
            || self.manage
            || (membership_type == MembershipType::Manager
                && !self.read_only
                && !self.hide_passwords),
    })
}
```

这里 manage 的三层逻辑：
1. **Admin/Owner** → 直接 true（角色压制）
2. **manage 标记** → 原始配置为 true 则 true
3. **Manager + 读写权限** → `!read_only && !hide_passwords` 时提升为 true

### 4.4 manage 标志的三处计算对比总结

| 计算场景 | Admin/Owner | cu.manage=true 的 User | Manager + 读写 | 普通User + 读写 |
|----------|-------------|----------------------|----------------|----------------|
| 用户同步 `to_json_details()` | true | **false** | true | false |
| 成员详情 `to_json_user_details()` | true | **true** | true | false |
| 集合成员 `to_json_details_for_member()` | true | **true** | true | false |

核心差异：**用户同步时普通 User 的 manage=true 被角色门槛过滤掉**，而在管理员视角和集合成员视角则保留原始配置。

### 4.5 CipherSyncData 中组权限的预合并

在 [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/ciphers.rs#L2175-L2193) 中，CipherSyncData 构建时对同一集合的多个组权限做了**预合并**：

```rust
let user_collections_groups: HashMap<CollectionId, CollectionGroup> =
    CollectionGroup::find_by_user(user_id, conn).await.into_iter().fold(
        HashMap::new(),
        |mut combined_permissions, cg| {
            combined_permissions
                .entry(cg.collections_uuid.clone())
                .and_modify(|existing| {
                    existing.read_only &= cg.read_only;        // AND: 任一组可写→可写
                    existing.hide_passwords &= cg.hide_passwords;  // AND: 任一组可见→可见
                    existing.manage |= cg.manage;              // OR: 任一组可管理→可管理
                })
                .or_insert(cg);
            combined_permissions
        },
    );
```

这意味着在 CipherSyncData 中，**每个集合只保留一个合并后的组权限记录**，而非多个组的原始记录。合并规则与 Cipher 的多集合合并完全一致。

---

## 五、group full access 为什么不再返回单独 collection

### 5.1 判定条件

在 [to_json_user_details()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L556-L562) 中：

```rust
let full_access_group = CONFIG.org_groups_enabled()
    && Group::is_in_full_access_group(&self.user_uuid, &self.org_uuid, conn).await;

let collections: Vec<Value> = if include_collections && !(full_access_group || self.access_all) {
    // 返回单独的集合列表
} else {
    Vec::with_capacity(0)  // 空！
};
```

[Group::is_in_full_access_group()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/group.rs#L269-L284) 检查用户是否属于**任何一个** `access_all = true` 的组：

```rust
pub async fn is_in_full_access_group(user_uuid: &UserId, org_uuid: &OrganizationId, conn: &DbConn) -> bool {
    // 查询: 用户所属组中，是否存在 access_all = true 的组
    groups::table
        .inner_join(groups_users::table...)
        .inner_join(users_organizations::table...)
        .filter(users_organizations::user_uuid.eq(user_uuid))
        .filter(groups::organizations_uuid.eq(org_uuid))
        .filter(groups::access_all.eq(true))
        .select(groups::access_all)
        .first::<bool>(conn)
        .unwrap_or_default()
}
```

### 5.2 不返回单独 collection 的完整原因

当 `full_access_group = true` 或 `self.access_all = true` 时，collections 为空数组。这是**设计上的有意选择**，不是遗漏：

**第一，避免客户端混淆**。Bitwarden 客户端根据 `accessAll` 字段决定权限模型：
- `accessAll: true` → 忽略 collections 列表，用户拥有所有集合的完整访问
- `accessAll: false` → 根据 collections 列表逐个判断权限

如果同时返回 `accessAll: true` 和非空 collections，客户端行为不确定。

**第二，组分配有独立端点**。成员详情 API 返回的 `groups` 字段已经列出了用户所属组的 ID。客户端通过**组详情端点** (`/organizations/<org_id>/groups/<group_id>`) 获取每个组关联的集合，不需要在成员级别重复。

**第三，减少数据冗余和不一致风险**。如果组内集合变更，而成员详情中的 collections 没同步更新，就会出现不一致。只保留组 ID 引用，让客户端按需查询，是更干净的做法。

### 5.3 同一用户既被直接分配又在 full_access 组中的处理

回到 [to_json_user_details()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L562)：

```rust
if include_collections && !(full_access_group || self.access_all)
```

**即使该用户有直接分配的集合**，只要 `full_access_group || self.access_all` 为 true，collections 就为空。因为 full_access 语义已经覆盖了所有集合，单独列出子集没有意义。

但在 `to_json_details()`（用户自己同步集合时），即使有 full_access，集合仍会出现在同步结果中（只是 readOnly/hidePasswords/manage 按 full_access 路径计算），因为客户端需要知道**哪些集合存在**。

---

## 六、共享条目（Cipher）可见性计算

### 6.1 访问限制计算入口

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

### 6.2 第一级：完全访问检查（短路返回）

如果满足以下任一条件，直接返回无限制访问 `(false, false, true)`：

```rust
if self.is_owned_by_user(user_uuid)                    // 1. 用户直接拥有
    || self.is_in_full_access_org(...)                 // 2. 组织级 full_access
    || self.is_in_full_access_group(...)               // 3. 组级 full_access
{
    return Some((false, false, true));
}
```

[is_in_full_access_org()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/cipher.rs#L558-L575) 检查用户在该组织中是否 `has_full_access()`（即 Admin/Owner 或 Manager+access_all=true）。

[is_in_full_access_group()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/cipher.rs#L577-L594) 检查 CipherSyncData 中的 `user_group_full_access_for_organizations`，这是在 [CipherSyncData::new()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/ciphers.rs#L2196-L2200) 中预计算的：

```rust
let user_group_full_access_for_organizations: HashSet<OrganizationId> =
    Group::get_orgs_by_user_with_full_access(user_id, conn).await.into_iter().collect();
```

**注意**：这里会检查组级 full_access！与 `to_json_details()` 不同！

### 6.3 第二级：集合权限收集（用户直接权限优先于组权限）

```rust
// 优先使用用户直接分配的权限
let user_permissions = self.get_user_collections_access_flags(...).await;
if user_permissions.is_empty() {
    // 如果没有直接分配，使用组分配的权限
    user_permissions = self.get_group_collections_access_flags(...).await;
} else {
    user_permissions
}
```

**重要：用户直接权限优先于组权限**。如果用户被直接分配了任一包含该条目的集合，则**只使用**用户直接权限，组权限被完全忽略。这是 `if/else` 结构，不是合并。

### 6.4 第三级：多集合权限合并

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

### 6.5 权限判定流程图

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

## 七、移除成员后的访问边界与同步通知

### 7.1 删除成员时的两层切断机制

删除成员时，Vaultwarden 通过**数据层清理 + 推送通知**两个维度确保访问被及时切断：

**维度一：数据层即时清理**

在 [delete_member_impl()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1677-L1719) 中，删除操作调用 `Membership::delete()`：

```rust
// organizations.rs L1714-L1718
if let Some(user) = User::find_by_uuid(&member_to_delete.user_uuid, conn).await {
    nt.send_user_update(UpdateType::SyncOrgKeys, &user, ...).await;
}
member_to_delete.delete(conn).await
```

[Membership::delete()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L772-L784) 的级联清理：

```rust
pub async fn delete(self, conn: &DbConn) -> EmptyResult {
    User::update_uuid_revision(&self.user_uuid, conn).await;        // ① 更新用户修订时间戳
    CollectionUser::delete_all_by_user_and_org(&self.user_uuid, &self.org_uuid, conn).await?;  // ② 删除集合分配
    GroupUser::delete_all_by_member(&self.uuid, conn).await?;       // ③ 删除组成员关系
    diesel::delete(users_organizations::table.filter(...)).execute(conn)  // ④ 删除成员记录
}
```

执行顺序的关键意义：
- ① 在 ②③④ **之前**执行，确保用户修订时间戳先更新
- ②③ 在 ④ **之前**执行，确保关联数据先清理，成员记录最后删除
- 即使推送通知因网络问题未送达，数据层的清理已经完成，被删除用户**下一次任何 API 请求**都会因成员记录不存在而被拒绝

**维度二：推送通知即时触发**

`nt.send_user_update(UpdateType::SyncOrgKeys, ...)` 在 `delete()` **之前**调用，通过两个通道推送：

1. **WebSocket 通知**（[notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/notifications.rs#L353-L355)）：
   ```rust
   if CONFIG.enable_websocket() {
       self.send_update(&user.uuid, &data).await;
   }
   ```
   向用户的所有活跃 WebSocket 连接推送 `SyncOrgKeys` 消息，客户端收到后立即触发全量同步。

2. **Push 推送**（[notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/notifications.rs#L357-L359)）：
   ```rust
   if CONFIG.push_enabled() {
       push_user_update(ut, user, push_uuid, conn).await;
   }
   ```
   通过 Bitwarden Push 服务向移动端推送通知，触发客户端同步。

### 7.2 修订时间戳（revision）的双重保障

[User::update_uuid_revision()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/user.rs#L351-L355) 更新 `users.updated_at` 时间戳：

```rust
pub async fn update_uuid_revision(uuid: &UserId, conn: &DbConn) {
    Self::update_revision_impl(uuid, &Utc::now().naive_utc(), conn).await;
}
```

客户端的同步机制依赖此时间戳：
1. 客户端每次同步时记录最新的 `revision_date`
2. 下次同步前先检查 `POST /accounts/revision-date`，如果服务器时间戳更新则触发全量同步
3. 即使 WebSocket/Push 都未送达，客户端的**定期轮询**也会发现时间戳变化并触发同步

### 7.3 确认成员时的同步通知

在 [confirm_invite_impl()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/organizations.rs#L1396-L1461) 中，确认成员后同样发送 `SyncOrgKeys`：

```rust
if let Some(user) = User::find_by_uuid(&member_to_confirm.user_uuid, conn).await {
    nt.send_user_update(UpdateType::SyncOrgKeys, &user, headers.device.push_uuid.as_ref(), conn).await;
}
```

这是因为确认成员后，用户的组织密钥（org key）发生了变化，客户端需要重新同步来获取组织加密密钥和解密共享条目。

### 7.4 其他触发 revision 更新的操作

以下操作都会通过 `User::update_uuid_revision()` 更新修订时间戳，间接触发客户端重新同步：

| 操作 | 触发位置 | 影响范围 |
|------|----------|----------|
| 删除成员 | [Membership::delete()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L773) | 被删除用户 |
| 保存成员 | [Membership::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L740) | 被修改用户 |
| 保存组织 | [Organization::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L344) | 组织内所有成员 |
| 保存集合 | [Collection::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L162) | 集合关联的所有成员 |
| 保存集合-用户关联 | [CollectionUser::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L682) | 被分配用户 |
| 保存条目 | [Cipher::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/cipher.rs#L442) | 条目关联的所有用户 |
| 保存组-用户关联 | [GroupUser::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/group.rs#L496) | 被修改的组成员 |
| 保存集合-组关联 | [CollectionGroup::save()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/group.rs#L322) | 组内所有用户 |

### 7.5 撤销（Revoke） vs 删除（Delete）的访问边界差异

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

| 操作 | 成员记录 | 集合分配 | 组成员 | 推送通知 | 可恢复 |
|------|----------|----------|--------|----------|--------|
| Revoke (撤销) | 保留，status 减 128 | **保留** | **保留** | 无 SyncOrgKeys | 是（restore） |
| Delete (删除) | 彻底删除 | 级联删除 | 级联删除 | **发送 SyncOrgKeys | 否 |

**撤销的访问切断方式不同**：撤销不删除任何数据，只修改 status 字段。访问切断完全依赖 `has_status(MembershipStatus::Confirmed)` 检查：

- [has_full_access()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L818)：要求 `has_status(Confirmed)`
- [can_access_collection()](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L150)：要求 `has_status(Confirmed)`
- [auth.rs 中的 Guard](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/auth.rs#L703)：`membership_status != Revoked`

**撤销的安全风险**：撤销操作**不发送 SyncOrgKeys 通知**，也不更新用户 revision 时间戳。被撤销用户的客户端可能不会立即感知到权限变更，直到下一次定期同步。在这段窗口期内，客户端缓存的数据仍可在本地查看（但无法与服务端交互，因为所有 API 请求都会被 Guard 拦截）。

### 7.6 访问切断的完整时序图

```
管理员发起删除成员请求
    ↓
delete_member_impl()
    ├── 权限校验（Only Owner can delete Admin/Owner）
    ├── 事件日志记录
    ├── nt.send_user_update(SyncOrgKeys) ← 推送通知先发
    │   ├── WebSocket → 客户端立即收到同步信号
    │   └── Push → 移动端收到推送
    └── Membership::delete()
        ├── User::update_uuid_revision() ← 修订时间戳更新
        ├── CollectionUser::delete_all_by_user_and_org() ← 集合分配清除
        ├── GroupUser::delete_all_by_member() ← 组成员关系清除
        └── DELETE users_organizations ← 成员记录删除

客户端收到 SyncOrgKeys 或下次轮询发现 revision 变化
    ↓
触发全量同步 /sync
    ↓
CipherSyncData::new() 重新构建
    ├── members: 该用户已不在组织中 → 不包含此组织的 Membership
    ├── user_collections: 集合分配已删除 → 空
    ├── user_collections_groups: 组分配已删除 → 空
    └── user_group_full_access_for_organizations: 组全访问已删除 → 不包含此组织
    ↓
所有组织条目从同步结果中消失
    ↓
客户端本地清除这些条目 → 访问完全切断
```

---

## 八、关键代码位置速查表

| 功能 | 文件 | 关键方法 |
|------|------|----------|
| 角色定义 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs) | `MembershipType` enum |
| full_access 判断 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L818) | `has_full_access()` |
| 集合详情权限（同步） | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L93) | `to_json_details()` |
| 成员详情权限（管理视角） | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L535) | `to_json_user_details()` |
| 集合成员权限 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L941) | `to_json_details_for_member()` |
| 集合可访问性 | [collection.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/collection.rs#L149) | `can_access_collection()` |
| 条目权限计算 | [cipher.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/cipher.rs#L601) | `get_access_restrictions()` |
| 组权限预合并 | [ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/core/ciphers.rs#L2175) | `CipherSyncData::new()` |
| 组 full_access 检查 | [group.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/group.rs#L269) | `is_in_full_access_group()` |
| 删除成员清理 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L772) | `Membership::delete()` |
| 成员撤销/恢复 | [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/organization.rs#L273) | `revoke()`, `restore()` |
| 推送通知 | [notifications.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/api/notifications.rs#L342) | `send_user_update()` |
| 修订时间戳更新 | [user.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/db/models/user.rs#L351) | `update_uuid_revision()` |
| 权限 Guard | [auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/2-vaultwarden/src/auth.rs) | `OrgHeaders`, `AdminHeaders` 等 |
