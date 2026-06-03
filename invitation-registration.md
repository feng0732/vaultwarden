# Vaultwarden 邀请与注册链路详解

**两条独立链路，容易混淆，本文拆开讲解。**

---

## 一、两条链路总览

| 链路 | 入口 | 目标 | 核心数据 | 状态流转 |
|---|---|---|---|---|
| **Admin 面板邀请** | [invite_user](src/api/admin.rs#L307-L335) | 创建用户账号（不绑定组织） | `users` 表 + `invitations` 表（邮件禁用时） | Invited → Enabled（注册完成即结束） |
| **组织成员加入** | [send_invite](src/api/core/organizations.rs#L1029-L1171) | 将用户加入特定组织 | `users_organizations` 表（Membership） | Invited → Accepted → Confirmed |

**关键区别**：
- Admin 邀请 = 「创建账号入口」
- 组织邀请 = 「加入组织流程」

---

## 二、链路 A：Admin 面板邀请

### 2.1 入口与目的

**API**：`POST /admin/invite`

**作用**：管理员在 Admin 面板主动创建用户。这不是"邀请加入某个组织"，而是"直接在系统中创建一个用户账号"。

### 2.2 完整流程

```
Admin 面板 → invite_user
     │
     ▼
1. 检查邮箱是否已存在
   → 已存在 → 409 Conflict 报错
   → 不存在 → 继续
     │
     ▼
2. 创建 User 记录（password_hash 为空）
     │
     ▼
3. generate_invite() 分支
   ├─ 邮件启用 → 发送 JWT 邀请邮件
   │   ├─ org_id = FAKE_ADMIN_UUID ("00000000-...")
   │   ├─ member_id = FAKE_ADMIN_UUID
   │   └─ 组织名 = CONFIG.invitation_org_name()
   └─ 邮件禁用 → 保存 Invitation 记录到 invitations 表
     │
     ▼
4. 保存 User 到数据库
```

**核心代码** [admin.rs#L309-L322](src/api/admin.rs#L309-L322)：
```rust
async fn generate_invite(user: &User, conn: &DbConn) -> EmptyResult {
    if CONFIG.mail_enabled() {
        let org_id: OrganizationId = if CONFIG.sso_enabled() {
            FAKE_SSO_IDENTIFIER.into()
        } else {
            FAKE_ADMIN_UUID.into()  // 关键点：用假的 UUID
        };
        let member_id: MembershipId = FAKE_ADMIN_UUID.to_owned().into();
        mail::send_invite(user, org_id, member_id, &CONFIG.invitation_org_name(), None).await
    } else {
        let invitation = Invitation::new(&user.email);
        invitation.save(conn).await
    }
}
```

### 2.3 邮件开关分支对比

| 环节 | 邮件启用 | 邮件禁用 |
|---|---|---|
| 凭证载体 | JWT Token（邮件链接） | `invitations` 表记录 |
| JWT org_id | FAKE_ADMIN_UUID | 不生成 JWT |
| JWT member_id | FAKE_ADMIN_UUID | 不生成 JWT |
| 用户操作 | 点击邮件链接 → 设置密码 → 完成注册 | 直接注册 → 系统自动消耗 Invitation |
| 状态变化 | User: password_hash 从空 → 有值 | 同左 |

### 2.4 用户注册完成

用户通过 Admin 邀请注册时，流程在 [register](src/api/core/accounts.rs#L172-L346) 中处理：

```rust
// 有 org_invite_token (邮件启用场景)
if let Some(token) = data.org_invite_token {
    let claims = decode_invite(&token)?;
    // 因为是 Admin 邀请，claims.org_id == FAKE_ADMIN_UUID
    // 不会触发 accept_org_invite 逻辑
}

// 邮件禁用场景：消耗 Invitation 记录
else if Invitation::take(&email, &conn).await {
    Membership::accept_user_invitations(&user.uuid, &conn).await?;
}
```

**注意**：Admin 邀请的 JWT 中 `org_id = FAKE_ADMIN_UUID`，这是一个特殊标记，在 `accept_invite` 中会被识别并跳过组织邀请逻辑。

### 2.5 Admin 邀请的策略检查时机

**Admin 邀请链路本身不做策略检查！**

因为 Admin 邀请只是创建用户账号，不关联任何组织，所以：
- 邀请时不检查 2FA 策略
- 邀请时不检查 SingleOrg 策略
- 后续用户加入组织时才会检查

---

## 三、链路 B：组织成员加入

### 3.1 入口与目的

**API**：`POST /organizations/<org_id>/users/invite`

**作用**：组织管理员邀请用户加入**某个具体组织**。这才是真正的"组织邀请"流程。

### 3.2 完整流程（三步状态流转 + 三条接受路径）

```
组织管理员 → send_invite
     │
     ▼
═══════════════════════════════════
  第一步：创建 Membership (Invited)
═══════════════════════════════════
     │
     ▼
1. 用户存在性检查分支
   ├─ 用户不存在
   │   ├─ 检查 invitations_allowed
   │   ├─ 检查邮箱域名白名单
   │   ├─ 邮件禁用 → 保存 Invitation 记录
   │   └─ 创建 stub User (password_hash 为空)
   └─ 用户已存在
       ├─ 检查是否已在组织中（任何状态都拒绝）
       └─ 邮件禁用 + 有密码 → 直接跳到 Accepted
     │
     ▼
2. 创建 Membership 记录
   ├─ user_uuid, org_uuid
   ├─ invited_by_email
   ├─ status = Invited(0) 或 Accepted(1)
   ├─ atype = 角色 (Owner/Admin/User/Manager)
   └─ access_all, collections, groups
     │
     ▼
3. 邮件启用 → 发送 JWT 邀请邮件
   ├─ org_id = 真实组织 ID
   ├─ member_id = 真实 Membership ID
   └─ 组织名 = 真实组织名称
     │
     ▼
═══════════════════════════════════
  第二步：用户接受 (Accepted) —— 三条路径
═══════════════════════════════════

     ┌──────────────────────────────┬──────────────────────────────┐
     │ 路径 A：JWT Token 接受        │ 路径 B：org_identifier 接受   │ 路径 C：邮件禁用自动接受
     │                              │                              │
     │ POST /organizations/<org_id>/│ POST /accounts/set-password  │ 注册时自动触发
     │ users/<member_id>/accept     │ （设置密码时）                 │
     │                              │                              │
     │ 1. 解码 JWT                  │ 1. 检查 org_identifier 字段   │ 1. Invitation::take()
     │ 2. 三重校验：                 │ 2. 排除 FAKE_ADMIN_UUID 和     │ 2. accept_user_invitations
     │    email/org_id/member_id    │    FAKE_SSO_IDENTIFIER       │
     │ 3. member_id == FAKE_ADMIN?  │ 3. 找到对应 Membership       │
     │    ├─ 是 → Admin 邀请        │ 4. accept_org_invite()        │
     │    └─ 否 → 组织邀请          │                              │
     │ 4. accept_org_invite         │                              │
     └──────────────────────────────┴──────────────────────────────┘
     │
     ▼
accept_org_invite 公共逻辑：
   ├─ 检查 status == Invited（防止重复接受）
   ├─ status → Accepted
   ├─ 设置 reset_password_key（如果启用了重置密码自动注册策略）
   ├─ 【策略检查】OrgPolicy::check_user_allowed
   └─ 发送"邀请已接受"通知给邀请者
     │
     ▼
═══════════════════════════════════
  第三步：管理员确认 (Confirmed)
═══════════════════════════════════
     │
     ▼
POST /organizations/<org_id>/users/<member_id>/confirm
     │
     ▼
1. 检查 status == Accepted
   → 不是 Accepted 则报错 "User in invalid state"
     │
     ▼
2. status → Confirmed
3. 设置 akey（组织加密密钥）
4. 【策略检查】OrgPolicy::check_user_allowed
5. 发送确认通知邮件
6. 通知用户同步组织密钥
```

### 3.3 接受邀请的三条路径详解

#### 路径 A：JWT Token 显式接受

**API**：`POST /organizations/<org_id>/users/<member_id>/accept`

**代码**：[accept_invite](src/api/core/organizations.rs#L1271-L1324)

关键点：
- URL 中的 org_id 和 member_id 只用于路由，真正校验的是 JWT claims 中的值
- 校验三重匹配：`claims.email == 当前用户`、`claims.org_id == URL org_id`、`claims.member_id == URL member_id`
- `member_id == FAKE_ADMIN_UUID` → Admin 邀请，跳过组织逻辑，发确认邮件
- 否则 → 调用 `accept_org_invite`

#### 路径 B：设置密码时通过 org_identifier 接受

**API**：`POST /accounts/set-password`

**代码**：[post_set_password](src/api/core/accounts.rs#L379-L392)

```rust
if let Some(identifier) = data.org_identifier
    && identifier != FAKE_SSO_IDENTIFIER
    && identifier != FAKE_ADMIN_UUID  // 排除 Admin 邀请
{
    let Some(org) = Organization::find_by_uuid(&identifier.into(), &conn).await else {
        err!("Failed to retrieve the associated organization")
    };

    // 通过用户+组织找到对应的 Membership
    let Some(membership) = Membership::find_by_user_and_org(&user.uuid, &org.uuid, &conn).await else {
        err!("Failed to retrieve the invitation")
    };

    accept_org_invite(&user, membership, None, &conn).await?;
}
```

**适用场景**：新用户首次设置密码时，前端传入 `org_identifier` 字段，直接接受对应组织的邀请。

#### 路径 C：邮件禁用时自动接受

**触发点**：
1. 注册时：[register](src/api/core/accounts.rs#L269-L271) → `accept_user_invitations`
2. 设置密码后：[post_set_password](src/api/core/accounts.rs#L396-L398) → `accept_user_invitations`

**代码**：[accept_user_invitations](src/db/models/organization.rs#L864-L871)

```rust
pub async fn accept_user_invitations(user_uuid: &UserId, conn: &DbConn) -> EmptyResult {
    diesel::update(users_organizations::table)
        .filter(users_organizations::user_uuid.eq(user_uuid))
        .filter(users_organizations::status.eq(MembershipStatus::Invited as i32))
        .set(users_organizations::status.eq(MembershipStatus::Accepted as i32))
        .execute(conn)
        .await?;
    Ok(())
}
```

**注意**：`accept_user_invitations` 是批量操作，将该用户所有 `Invited` 状态的 Membership 转为 `Accepted`，但**不做策略检查**（因为没有每条 Membership 单独调用 `accept_org_invite`）。

### 3.4 邮件开关分支对比

| 环节 | 邮件启用 | 邮件禁用 |
|---|---|---|
| 新用户邀请 | 发送含真实 org_id/member_id 的 JWT 邮件 | 存 `invitations` 表记录 |
| 已有用户邀请 | 发送邀请邮件，status = Invited | 若用户有密码则 status = Accepted（跳过第二步） |
| 用户接受 | 路径 A：点击邮件链接，显式调用 accept | 路径 C：注册时自动 accept_user_invitations |
| 管理员确认 | 必须手动 confirm | 同左 |
| Invitation 表 | 不使用 | 作为"准入令牌"，注册时 take 消耗 |

**关键代码** [organizations.rs#L1092-L1095](src/api/core/organizations.rs#L1092-L1095)：
```rust
// 邮件禁用 + 用户已有密码 → 自动跳到 Accepted
if !CONFIG.mail_enabled() && !user.password_hash.is_empty() {
    member_status = MembershipStatus::Accepted as i32;
}
```

### 3.5 重复邀请防护

在 [send_invite](src/api/core/organizations.rs#L1088-L1090) 中：
```rust
if Membership::find_by_user_and_org(&user.uuid, &org_id, &conn).await.is_some() {
    err!(format!("User already in organization: {email}"))
}
```

**注意**：`find_by_user_and_org` **不过滤状态**，即：
- Invited 状态 → 拒绝重复邀请
- Accepted 状态 → 拒绝重复邀请
- Confirmed 状态 → 拒绝重复邀请
- Revoked 状态 → 拒绝重复邀请！（这是一个设计选择：被撤销的成员也不能通过"重新邀请"回来，必须 restore）

### 3.6 策略检查时机

[OrgPolicy::check_user_allowed](src/db/models/org_policy.rs#L283-L317) 在组织邀请链路上被调用 **5 次**：

| 调用点 | 时机 | 检查内容 |
|---|---|---|
| [accept_org_invite](src/api/core/mod.rs#L287) | 用户接受邀请时（路径 A/B） | 2FA、SingleOrg |
| [confirm_invite_impl](src/api/core/organizations.rs#L1427) | 管理员确认时 | 2FA、SingleOrg |
| [edit_member](src/api/core/organizations.rs#L1582) | 编辑成员时 | 2FA、SingleOrg |
| [restore_member_impl](src/api/core/organizations.rs#L2402) | 恢复成员时 | 2FA、SingleOrg |
| [admin::update_membership_type](src/api/admin.rs#L571) | Admin 改角色时 | 2FA、SingleOrg |

**⚠️ 路径 C 例外**：`accept_user_invitations` 是批量更新，**不调用策略检查**。这是一个边界：邮件禁用场景下自动接受的邀请，跳过了 2FA 和 SingleOrg 检查。

**策略检查前置条件**（[org_policy.rs#L284](src/db/models/org_policy.rs#L284)）：
```rust
if m.atype < MembershipType::Admin && m.status > (MembershipStatus::Invited as i32) {
    // 只对非 Admin、状态 > Invited 的成员检查
}
```

这意味着：
- **Admin/Owner 豁免**：不检查 2FA 和 SingleOrg
- **Invited 状态豁免**：接受邀请前不检查（Invited=0，0 > 0 为假）

**检查内容**：
1. **2FA 要求**：组织启用 2FA 策略，但用户未设置 2FA → 拒绝
   - 除非 `email_2fa_auto_fallback` 启用，则自动激活邮箱 2FA
2. **跨组织 SingleOrg**：用户已属于另一个启用 SingleOrg 的组织 → 拒绝
3. **本组织 SingleOrg**：当前组织启用 SingleOrg，但用户已加入其他组织 → 拒绝

---

## 四、撤销与恢复设计

### 4.1 为什么用 status ± 128 而不是布尔字段？

代码注释 [organization.rs#L249-L253](src/db/models/organization.rs#L249-L253)：
```rust
// Used to either subtract or add to the current status
// The number 128 should be fine, it is well within the range of an i32
// The same goes for the database where we only use INTEGER
// It should also provide enough room for 100+ types
const ACTIVATE_REVOKE_DIFF: i32 = 128;
```

**设计意图**：
1. **可逆且保留原始状态**：撤销不是设为固定值 -1，而是 `status -= 128`。这样恢复时 `status += 128` 就能精确回到撤销前的状态。

   例子：
   ```
   Invited(0)   → revoke → -128 → restore → Invited(0)
   Accepted(1)  → revoke → -127 → restore → Accepted(1)
   Confirmed(2) → revoke → -126 → restore → Confirmed(2)
   ```

2. **状态判断简单**：任何 `status < 0` 都是被撤销的，任何 `status >= 0` 都是有效的。

3. **可扩展**：128 留出了足够空间，即使未来增加 100 种状态也不会冲突。

### 4.2 撤销流程

**API**：`PUT /organizations/<org_id>/users/<member_id>/revoke`

**入口**：[revoke_member_impl](src/api/core/organizations.rs#L2281-L2322)

```
撤销前置检查：
1. 不能撤销自己
2. 撤销 Owner 需要自己是 Owner
3. 不能撤销最后一个 Owner（组织必须至少有一个确认的 Owner）
4. 状态 > Revoked(-1) 才能撤销

执行：
member.revoke()  // status -= 128
member.save()
记录日志
```

**注意**：撤销时**不做策略检查**，只做权限检查。

### 4.3 恢复流程：对 Active Invited 的边界

**API**：`PUT /organizations/<org_id>/users/<member_id>/restore`

**入口**：[restore_member_impl](src/api/core/organizations.rs#L2381-L2420)

**两层边界检查**：

```rust
// 第一层：进入分支的条件
match Membership::find_by_uuid_and_org(member_id, org_id, conn).await {
    Some(mut member) if member.status < MembershipStatus::Accepted as i32 => {
        // status < 1 才能进入这个分支
        // 包括: -128, -127, -126, 0 (Invited!)
        ...
        member.restore();  // 第二层检查在 restore() 内部
        ...
    }
    Some(_) => err!("User is already active"),
    None => err!("User not found in organization"),
}
```

**restore() 内部的第二层检查** [organization.rs#L273-L279](src/db/models/organization.rs#L273-L279)：
```rust
pub fn restore(&mut self) -> bool {
    if self.status < MembershipStatus::Invited as i32 {
        // status < 0 才真正执行恢复
        self.status += ACTIVATE_REVOKE_DIFF;
        return true;
    }
    false
}
```

**边界分析表**：

| 状态值 | 状态含义 | 第一层检查 (status < 1) | 第二层检查 (status < 0) | 结果 |
|---|---|---|---|---|
| -128 | Revoked Invited | ✅ 进入 | ✅ 执行 restore | 恢复到 Invited(0) |
| -127 | Revoked Accepted | ✅ 进入 | ✅ 执行 restore | 恢复到 Accepted(1) |
| -126 | Revoked Confirmed | ✅ 进入 | ✅ 执行 restore | 恢复到 Confirmed(2) |
| 0 | Active Invited | ✅ 进入 | ❌ 不执行 restore | restore 返回 false，无报错 |
| 1 | Active Accepted | ❌ 不进入 | N/A | 直接报错 "User is already active" |
| 2 | Active Confirmed | ❌ 不进入 | N/A | 直接报错 "User is already active" |

**⚠️ 边界行为**：Active Invited 状态（status=0）的成员调用 restore API：
- 第一层检查通过，进入分支
- 但 restore() 内部不执行任何操作（返回 false）
- 不会报错，也不会记录日志
- 最终效果：静默的空操作

**恢复前置检查**（进入分支后）：
1. 不能恢复自己
2. 恢复 Owner 需要自己是 Owner
3. `member.restore()` 执行（status += 128）
4. 【策略检查】`OrgPolicy::check_user_allowed` → 恢复后验证当前状态是否符合策略
5. 保存并记录日志

**关键点**：恢复时做策略检查！因为用户在被撤销期间可能：
- 关闭了 2FA → 恢复时如果组织要求 2FA 就会失败
- 加入了另一个 SingleOrg 组织 → 恢复时会失败

这就是为什么注释说 [organizations.rs#L2400-L2401](src/api/core/organizations.rs#L2400-L2401)：
```rust
// This check need to be done after restoring to work with the correct status
```

必须先恢复状态，再用正确的状态做策略检查。

---

## 五、两条链路对比表

| 维度 | Admin 面板邀请 | 组织成员邀请 |
|---|---|---|
| **入口** | `/admin/invite` | `/organizations/<org_id>/users/invite` |
| **核心数据** | `users` + `invitations` 表 | `users_organizations` (Membership) |
| **JWT org_id** | FAKE_ADMIN_UUID | 真实组织 ID |
| **JWT member_id** | FAKE_ADMIN_UUID | 真实 Membership ID |
| **状态流转** | 1 步：设置密码完成注册 | 3 步：Invited → Accepted → Confirmed |
| **接受邀请路径** | N/A（无组织邀请） | 三条路径：JWT / org_identifier / 自动 accept |
| **邀请时策略检查** | ❌ 不检查 | ❌ 不检查（Invited 状态豁免） |
| **接受时策略检查** | ❌ 没有"接受"概念 | ✅ 路径 A/B 检查，路径 C 不检查 |
| **确认时策略检查** | ❌ 没有"确认"概念 | ✅ check_user_allowed |
| **恢复时策略检查** | N/A（无 Membership） | ✅ check_user_allowed |
| **撤销机制** | N/A（删除用户） | status - 128，可恢复 |
| **重复邀请防护** | 邮箱已存在就报错 | 任何状态 Membership 都拒绝 |
| **邮件禁用时** | 存 Invitation 表 | 已有密码用户直接 Accepted |

---

## 六、容易混淆的点

### 6.1 "Invitation" 这个词指的是什么？

代码中有三种不同的 "Invitation"：

1. **Invitation 表** [user.rs#L74-L79](src/db/models/user.rs#L74-L79)
   - 只在邮件禁用时使用
   - 只是一个"邮箱被邀请过"的标记
   - 不含组织信息
   - 注册后立即删除

2. **Membership status = Invited**
   - 组织邀请流程的第一步状态
   - 有真实的 org_id 和 member_id

3. **JWT Invite Token**
   - 邮件中的链接凭证
   - 可能是 Admin 邀请（org_id=FAKE_ADMIN_UUID）
   - 可能是组织邀请（org_id=真实 ID）

### 6.2 为什么 restore 要做策略检查，revoke 不做？

**Revoke 是管理员的主动权限操作**：
- 管理员说"这个人不能访问了" → 直接执行
- 不需要检查用户的 2FA 或其他组织归属
- 是一种"阻断"操作

**Restore 是重新授予访问权限**：
- 相当于重新"加入"组织
- 撤销期间用户的状态可能变了（关了 2FA、加入了其他组织）
- 需要重新验证是否符合当前组织策略
- 是一种"准入"操作

### 6.3 为什么重复邀请连 Revoked 状态也拒绝？

设计选择：被撤销的成员应该用 restore 来恢复，而不是重新邀请。

理由：
1. 保留历史记录（谁邀请的、什么时候加入的）
2. 保留权限配置（collections、groups、role）
3. 避免意外创建重复 Membership

如果确实想"重新开始"，需要先删除 Membership，再邀请。

### 6.4 Active Invited 调用 restore 会怎样？

静默的空操作！不报错，也不改变状态。这是一个边界行为：
- 第一层条件 `status < 1` 通过
- 第二层条件 `status < 0` 不通过
- restore() 返回 false，不执行操作
- 不调用策略检查，不记录日志

---

## 七、关键代码位置速查

| 功能 | 文件 | 行号 |
|---|---|---|
| Admin 邀请入口 | [admin.rs](src/api/admin.rs) | L307-L335 |
| 组织邀请入口 | [organizations.rs](src/api/core/organizations.rs) | L1029-L1171 |
| JWT 接受邀请 | [organizations.rs](src/api/core/organizations.rs) | L1271-L1324 |
| org_identifier 接受邀请 | [accounts.rs](src/api/core/accounts.rs) | L379-L392 |
| 接受组织邀请公共逻辑 | [mod.rs](src/api/core/mod.rs) | L273-L301 |
| 自动批量接受邀请 | [organization.rs](src/db/models/organization.rs) | L864-L871 |
| 确认邀请 | [organizations.rs](src/api/core/organizations.rs) | L1396-L1461 |
| 撤销成员 | [organizations.rs](src/api/core/organizations.rs) | L2281-L2322 |
| 恢复成员 | [organizations.rs](src/api/core/organizations.rs) | L2381-L2420 |
| 策略检查 | [org_policy.rs](src/db/models/org_policy.rs) | L283-L317 |
| Membership 状态枚举 | [organization.rs](src/db/models/organization.rs) | L74-L80 |
| Membership restore 方法 | [organization.rs](src/db/models/organization.rs) | L273-L279 |
| 撤销/恢复常量 | [organization.rs](src/db/models/organization.rs) | L253 |
