# Vaultwarden 邀请与注册链路详解

## 一、核心数据模型

### 1.1 两张关键表

| 表 | 模型 | 主键 | 作用 |
|---|---|---|---|
| `invitations` | [Invitation](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/db/models/user.rs#L74-L79) | `email` | 临时标记"此邮箱被邀请"，仅邮件禁用时使用，注册后立即删除 |
| `users_organizations` | [Membership](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/db/models/organization.rs#L43-L60) | `uuid` (MembershipId) | 用户与组织的关系记录，承载邀请全生命周期状态 |

> **要点**：`invitations` 表只是一个"准入令牌"，不含组织信息；真正的邀请关系存储在 `users_organizations` 表的 `Membership` 记录中。

### 1.2 Membership 状态枚举

定义于 [MembershipStatus](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/db/models/organization.rs#L74-L80)：

```
Revoked  = -1   // 被撤销（实际存储为 status - 128）
Invited  =  0   // 已邀请，等待用户接受
Accepted =  1   // 用户已接受，等待管理员确认
Confirmed = 2   // 管理员已确认，正式成员
```

**撤销机制**：不是直接设为 -1，而是 `status -= 128`，这样可以恢复到撤销前的状态（`status += 128`）。见 [ACTIVATE_REVOKE_DIFF](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/db/models/organization.rs#L253)。

### 1.3 Membership 类型枚举

定义于 [MembershipType](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/db/models/organization.rs#L95-L101)：

```
Owner   = 0   // 最高权限
Admin   = 1
User    = 2
Manager = 3   // Custom (4) 在内部被映射为 Manager
```

### 1.4 User 状态（隐式）

User 本身没有显式 status 字段，而是通过 `password_hash` 推导：

```rust
// user.rs#L265-L269
let status = if self.password_hash.is_empty() {
    UserStatus::Invited   // 1 — 尚未设置密码
} else {
    UserStatus::Enabled   // 0 — 已完成注册
};
```

---

## 二、邀请状态流转全景

```
                    ┌──────────────────────────────────────────────────────┐
                    │              组织管理员发起邀请                        │
                    │     POST /organizations/<org_id>/users/invite        │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │          用户是否已存在？                               │
                    └─────┬──────────────────────────┬────────────────────┘
                          │                          │
                     不存在                        已存在
                          │                          │
               ┌──────────▼──────────┐    ┌──────────▼──────────┐
               │ 创建 User (password │    │ 检查是否已在组织中    │
               │ _hash 为空)         │    │ → 已在则报错          │
               │                     │    └──────────┬──────────┘
               │ 邮件禁用时:          │               │
               │  保存 Invitation    │          不在组织中
               │ 邮件启用时:          │               │
               │  发送邀请邮件(JWT)   │               │
               └──────────┬──────────┘               │
                          │                          │
                          │    ┌─────────────────────┘
                          │    │
                          │    │  邮件禁用 && 用户有密码？
                          │    │  → 自动 Accepted (跳过邀请)
                          │    │  否则 → Invited
                          │    │
               ┌──────────▼────▼───────────────────────────────────────┐
               │        Membership 记录创建                               │
               │   status = Invited(0) 或 Accepted(1)                    │
               └──────────────────────┬──────────────────────────────────┘
                                      │
           ┌──────────────────────────┼──────────────────────────────────┐
           │                          │                                  │
     邮件启用时                   邮件禁用 + 已有密码               邮件禁用 + 无密码
           │                          │                                  │
   用户收到邮件，点击            自动 Accepted                  用户直接注册
   链接接受邀请                      │                            (见注册流程)
           │                          │                                  │
   POST .../accept                   │                                  │
           │                          │                                  │
   status → Accepted(1) ◄────────────┘◄─────────────────────────────────┘
           │
           │  管理员确认
           │  POST .../confirm
           │
   status → Confirmed(2)
           │
     ══════╧═══════════
     正式组织成员
```

---

## 三、邀请发起流程详解

入口：[send_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1029-L1171)

### 3.1 前置校验

1. **权限**：非 Owner 不能邀请 Manager/Admin/Owner（[L1052](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1052-L1054)）
2. **集合和分组归属**：验证传入的 collection/group 确实属于该组织

### 3.2 用户存在性分支

```rust
// organizations.rs#L1068-L1098
let user = match User::find_by_mail(email, &conn).await {
    None => {
        // 用户不存在
        if !CONFIG.invitations_allowed() {
            err!("User does not exist: {email}")  // 不允许邀请时拒绝
        }
        if !CONFIG.is_email_domain_allowed(email) {
            err!("Email domain not eligible for invitations")
        }
        if !CONFIG.mail_enabled() {
            Invitation::new(email).save(&conn).await?;  // 邮件禁用：存 invitations 表
        }
        let mut new_user = User::new(email, None);
        new_user.save(&conn).await?;  // 创建 stub 用户（无密码）
        user_created = true;
        new_user
    }
    Some(user) => {
        // 用户已存在
        if Membership::find_by_user_and_org(&user.uuid, &org_id, &conn).await.is_some() {
            err!("User already in organization: {email}")  // 重复邀请防护
        }
        // 邮件禁用 + 用户有密码 → 自动跳到 Accepted
        if !CONFIG.mail_enabled() && !user.password_hash.is_empty() {
            member_status = MembershipStatus::Accepted as i32;
        }
        user
    }
};
```

### 3.3 创建 Membership

```rust
// organizations.rs#L1100-L1104
let mut new_member = Membership::new(user.uuid.clone(), org_id.clone(), Some(headers.user.email.clone()));
new_member.access_all = access_all;
new_member.atype = new_type;
new_member.status = member_status;  // Invited 或 Accepted
new_member.save(&conn).await?;
```

### 3.4 发送邀请邮件（邮件启用时）

调用 [mail::send_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/mail.rs#L282-L321)，生成包含 JWT token 的邀请链接。JWT claims 结构（[InviteJwtClaims](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/auth.rs#L294-L308)）：

```rust
pub struct InviteJwtClaims {
    pub sub: UserId,          // 被邀请用户 ID
    pub email: String,        // 被邀请用户邮箱
    pub org_id: OrganizationId,
    pub member_id: MembershipId,
    pub invited_by_email: Option<String>,
}
```

**发送失败回滚**：如果邮件发送失败，会删除刚创建的用户或 Membership 记录。

---

## 四、接受邀请流程

### 4.1 已注册用户接受邀请

入口：[accept_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1271-L1324)

```
POST /organizations/<org_id>/users/<member_id>/accept
Body: { token: "<JWT>", reset_password_key: "..." }
```

**校验链**：
1. 解码 JWT，验证 `claims.email == headers.user.email`（防止代接受）
2. 验证 `claims.org_id == URL中的org_id`
3. 验证 `claims.member_id == URL中的member_id`
4. 删除 `invitations` 表记录（`Invitation::take`）
5. 判断是否来自 Admin 面板（`member_id == FAKE_ADMIN_UUID`）：
   - **来自 Admin 面板**：发送确认邮件（自动确认）
   - **来自组织邀请**：调用 `accept_org_invite`

核心函数 [accept_org_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/mod.rs#L273-L301)：

```rust
async fn accept_org_invite(user, mut member, reset_password_key, conn) -> EmptyResult {
    if member.status != MembershipStatus::Invited as i32 {
        err!("User already accepted the invitation");  // 防止重复接受
    }
    member.status = MembershipStatus::Accepted as i32;
    member.reset_password_key = reset_password_key;
    OrgPolicy::check_user_allowed(&member, "join", conn).await?;  // 策略检查
    member.save(conn).await?;
    // 发送 "邀请已接受" 通知给邀请者
}
```

### 4.2 新用户通过注册接受邀请

入口：[register](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/accounts.rs#L172-L346)

**场景 A — 有 org_invite_token（通过邀请邮件注册）**：

```rust
// accounts.rs#L260-L268
if let Some(token) = data.org_invite_token {
    let claims = decode_invite(&token)?;
    if claims.email == email {
        email_verified = true;  // 邀请链接自动验证邮箱
        user
    }
}
```

注册完成后，不会自动接受邀请。用户需要后续通过 `accept_invite` 端点或 `post_set_password` 端点完成接受。

**场景 B — 有 Invitation 记录（邮件禁用时的邀请）**：

```rust
// accounts.rs#L269-L271
} else if Invitation::take(&email, &conn).await {
    Membership::accept_user_invitations(&user.uuid, &conn).await?;
    user
}
```

`accept_user_invitations` 会将该用户所有 `Invited` 状态的 Membership 批量更新为 `Accepted`。

### 4.3 设置密码时接受邀请

入口：[post_set_password](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/accounts.rs#L348-L409)

```rust
// accounts.rs#L379-L392
if let Some(identifier) = data.org_identifier {
    // 通过组织标识符找到对应的 Membership
    let Some(membership) = Membership::find_by_user_and_org(&user.uuid, &org.uuid, &conn).await else {
        err!("Failed to retrieve the invitation")
    };
    accept_org_invite(&user, membership, None, &conn).await?;
}

// 邮件禁用时：自动接受所有待处理邀请
if !CONFIG.mail_enabled() {
    Membership::accept_user_invitations(&user.uuid, &conn).await?;
}
```

---

## 五、管理员确认流程

入口：[confirm_invite_impl](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1396-L1461)

```
POST /organizations/<org_id>/users/<member_id>/confirm
Body: { id: "<member_id>", key: "<encrypted_key>" }
```

**前置条件**：
- Membership 状态必须为 `Accepted`（不是 Invited 也不是 Confirmed）
- 非 Owner 不能确认 Manager/Admin/Owner

**操作**：
1. `status = Confirmed`
2. 设置 `akey`（组织加密密钥，用用户公钥加密）
3. 执行策略检查（2FA 要求、SingleOrg 限制）
4. 发送确认通知邮件
5. 通知用户同步组织密钥

---

## 六、Admin 面板邀请

入口：[invite_user](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/admin.rs#L307-L335)

Admin 面板的邀请不关联特定组织，使用 `FAKE_ADMIN_UUID`（`"00000000-0000-0000-0000-000000000000"`）作为 org_id 和 member_id：

```rust
// admin.rs#L309-L322
async fn generate_invite(user: &User, conn: &DbConn) -> EmptyResult {
    if CONFIG.mail_enabled() {
        let org_id: OrganizationId = if CONFIG.sso_enabled() {
            FAKE_SSO_IDENTIFIER.into()
        } else {
            FAKE_ADMIN_UUID.into()
        };
        let member_id: MembershipId = FAKE_ADMIN_UUID.to_owned().into();
        mail::send_invite(user, org_id, member_id, &CONFIG.invitation_org_name(), None).await
    } else {
        let invitation = Invitation::new(&user.email);
        invitation.save(conn).await
    }
}
```

**特殊处理**：在 `accept_invite` 中，如果 `member_id == FAKE_ADMIN_UUID`，跳过组织邀请逻辑，直接发送确认邮件。

---

## 七、重新邀请

入口：[reinvite_member_impl](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1221-L1262)

```
POST /organizations/<org_id>/users/<member_id>/reinvite
```

**条件**：Membership 状态必须为 `Invited`。

**行为分支**：
- 邮件启用 → 重新发送邀请邮件
- 邮件禁用 + 用户无密码 → 保存 Invitation 记录
- 邮件禁用 + 用户有密码 → 删除 Invitation 记录，自动设为 Accepted

---

## 八、重复邀请边界场景

### 8.1 同一用户邀请到同一组织

**防护位置**：[send_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1088-L1089)

```rust
if Membership::find_by_user_and_org(&user.uuid, &org_id, &conn).await.is_some() {
    err!(format!("User already in organization: {email}"))
}
```

`find_by_user_and_org` **不过滤状态**，即无论用户处于 Invited / Accepted / Confirmed / Revoked 状态，都会拒绝重复邀请。

### 8.2 邀请已被接受后再次接受

**防护位置**：[accept_org_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/mod.rs#L279-L281)

```rust
if member.status != MembershipStatus::Invited as i32 {
    err!("User already accepted the invitation");
}
```

只有 `Invited` 状态才能转为 `Accepted`。

### 8.3 确认非 Accepted 状态的成员

**防护位置**：[confirm_invite_impl](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1419-L1421)

```rust
if member_to_confirm.status != MembershipStatus::Accepted as i32 {
    err!("User in invalid state")
}
```

只有 `Accepted` 状态才能转为 `Confirmed`。

### 8.4 邀请邮箱与当前登录用户不匹配

**防护位置**：[accept_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1284-L1285)

```rust
if !claims.email.eq(&headers.user.email) {
    err!("Invitation was issued to a different account")
}
```

防止用户 A 代用户 B 接受邀请。

### 8.5 用户注册时邮箱已被邀请（stub 用户存在）

**场景**：管理员邀请了 alice@example.com，创建了 stub User（password_hash 为空）。Alice 尝试自行注册。

**处理**：[register](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/accounts.rs#L254-L293)

```rust
match User::find_by_mail(&email, &conn).await {
    Some(user) => {
        if !user.password_hash.is_empty() {
            err!("Registration not allowed or user already exists")  // 已完成注册
        }
        // 密码为空 = stub 用户，允许注册覆盖
        if let Some(token) = data.org_invite_token {
            // 验证邀请 token
        } else if Invitation::take(&email, &conn).await {
            // 消耗 invitation 记录
            Membership::accept_user_invitations(...)
        } else if CONFIG.is_signup_allowed(&email) {
            // 普通注册
        } else {
            err!("Registration not allowed or user already exists")
        }
    }
    None => { /* 新用户注册 */ }
}
```

### 8.6 邮件发送失败时的回滚

**位置**：[send_invite](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/organizations.rs#L1123-L1130)

```rust
if let Err(e) = mail::send_invite(...).await {
    if user_created {
        user.delete(&conn).await?;   // 删除刚创建的 stub 用户
    } else {
        new_member.delete(&conn).await?;  // 删除刚创建的 Membership
    }
    err!(format!("Error sending invite: {e:?}"));
}
```

### 8.7 策略冲突导致的邀请失败

[OrgPolicy::check_user_allowed](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/db/models/org_policy.rs#L283-L311) 在以下节点被调用：
- `accept_org_invite` — 接受邀请时
- `confirm_invite_impl` — 确认邀请时
- `edit_member` — 编辑成员时
- `update_membership_type` — Admin 面板修改类型时

**检查内容**：
1. **2FA 要求**：组织启用了 2FA 策略，但用户未设置 2FA → 拒绝（除非 email_2fa_auto_fallback 启用）
2. **SingleOrg 策略**：用户已属于另一个启用了 SingleOrg 的组织 → 拒绝加入新组织
3. **当前组织的 SingleOrg**：用户已加入其他组织 → 拒绝

---

## 九、邮件启用 vs 禁用的影响对比

| 环节 | 邮件启用 | 邮件禁用 |
|---|---|---|
| 邀请新用户 | 发送含 JWT 的邀请邮件 | 存 `invitations` 表记录 |
| 已有用户被邀请 | 发送邀请邮件，status = Invited | 若用户有密码则 status = Accepted |
| 接受邀请 | 用户点击邮件链接，显式 accept | 注册时自动 `accept_user_invitations` |
| 确认邀请 | 管理员手动 confirm | 同左 |
| Invitation 表 | 不使用 | 作为"准入令牌"，注册时 `take` 消耗 |
| 注册验证 | 需邮箱验证 | 自动验证 |

---

## 十、关键配置项

| 配置 | 默认值 | 说明 |
|---|---|---|
| `signups_allowed` | true | 是否允许自由注册 |
| `signups_domains_whitelist` | 空 | 注册邮箱域名白名单 |
| `signups_verify` | false | 注册是否需要邮箱验证 |
| `invitations_allowed` | true | 组织管理员是否可以邀请用户（即使关闭自由注册） |
| `invitation_expiration_hours` | 120 | 邀请 JWT 过期时间（小时） |
| `invitation_org_name` | "Vaultwarden" | Admin 面板邀请邮件中的组织名 |
| `emergency_access_allowed` | true | 是否允许紧急访问 |

**注册准入优先级**（[register](file:///d:/fz/0601/solo-dogfeeding/code/8-vaultwarden/src/api/core/accounts.rs#L282-L293)）：
1. Invitation 记录存在（最高优先级，管理员邀请 > 自由注册限制）
2. `is_signup_allowed` 为 true
3. 紧急访问邀请存在
4. 以上都不满足 → 拒绝注册

---

## 十一、完整状态机

```
                 ┌────────────┐
                 │  不存在     │
                 └─────┬──────┘
                       │ 管理员邀请
                       ▼
                 ┌────────────┐
          ┌─────│  Invited(0) │
          │     └─────┬──────┘
          │           │ 用户接受
          │           ▼
          │     ┌────────────┐
          │     │ Accepted(1) │
          │     └─────┬──────┘
          │           │ 管理员确认
          │           ▼
          │     ┌─────────────┐
          │     │ Confirmed(2) │◄──────── 正式成员
          │     └─────┬───────┘
          │           │ 撤销
          │           ▼
          │     ┌────────────┐
          │     │ Revoked(-1) │  (实际: status - 128)
          │     └─────┬──────┘
          │           │ 恢复
          │           ▼
          │     ┌─────────────┐
          └────►│ Confirmed(2) │  (实际: status + 128)
                └─────────────┘

 注意：Invited/Accepted 也可被撤销和恢复
       撤销是通用的 status-128 操作，恢复是 status+128
```

**跳转路径**：
- 邮件禁用 + 已有密码：`Invited` → 直接 `Accepted`（跳过用户确认）
- Admin 面板邀请 + 邮件启用：`Invited` → 自动 `Confirmed`（用户注册即确认）
- 注册验证邮件流程：`register_verification_email` → `register_finish` → 可选带 `org_invite_token`
