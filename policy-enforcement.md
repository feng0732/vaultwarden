# Vaultwarden 组织策略执行机制分析

## 1. 策略数据模型

### 1.1 策略类型定义

策略类型定义在 [org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L31-L52) 的 `OrgPolicyType` 枚举中：

| 策略类型 | 值 | 服务端强制执行 | 说明 |
|---------|----|--------------|------|
| TwoFactorAuthentication | 0 | ✅ 是 | 双因素认证强制策略 |
| MasterPassword | 1 | ⚠️ 部分 | 主密码复杂度策略（服务端合并后返回客户端，无强制执行） |
| PasswordGenerator | 2 | ❌ 否 | 密码生成器策略（仅客户端执行） |
| SingleOrg | 3 | ✅ 是 | 单一组织限制策略 |
| PersonalOwnership | 5 | ✅ 是 | 个人所有权限制策略 |
| DisableSend | 6 | ✅ 是 | 禁用 Send 功能策略 |
| SendOptions | 7 | ✅ 是 | Send 选项策略 |
| ResetPassword | 8 | ✅ 是 | 密码重置策略 |
| RemoveUnlockWithPin | 14 | ❌ 否 | 移除 PIN 解锁策略（仅客户端执行） |
| RestrictedItemTypes | 15 | ❌ 否 | 限制项目类型策略（仅客户端执行） |
| UriMatchDefaults | 16 | ❌ 否 | URI 匹配默认值策略（仅客户端执行） |

### 1.2 策略数据结构

策略表结构定义在 [org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L18-L27)：

```rust
pub struct OrgPolicy {
    pub uuid: OrgPolicyId,      // 策略唯一ID
    pub org_uuid: OrganizationId, // 所属组织ID
    pub atype: i32,             // 策略类型（对应OrgPolicyType）
    pub enabled: bool,          // 是否启用
    pub data: String,           // JSON格式的策略配置数据
}
```

## 2. 策略范围与角色约束

### 2.1 成员状态定义

成员状态定义在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/organization.rs#L75-L80)：

| 状态 | 值 | 说明 | 是否受策略约束 |
|------|----|------|--------------|
| Revoked | -1 | 已撤销 | ❌ 否 |
| Invited | 0 | 已邀请 | ❌ 否 |
| Accepted | 1 | 已接受 | ✅ 是（部分策略） |
| Confirmed | 2 | 已确认 | ✅ 是 |

**重要：** 不同策略对成员状态的过滤条件不同，详见各策略分析。

### 2.2 角色层级与比较逻辑

角色类型定义在 [organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/organization.rs#L96-L101)：

| 角色 | 枚举值 | ACCESS_LEVEL | 说明 | 是否受策略约束 |
|------|--------|-------------|------|--------------|
| Owner | 0 | 3 | 所有者 | ❌ 否 |
| Admin | 1 | 2 | 管理员 | ❌ 否 |
| User | 2 | 0 | 普通用户 | ✅ 是 |
| Manager | 3 | 1 | 经理 | ✅ 是 |

**角色比较机制**：
- 直接数值比较：`Owner(0) < Admin(1) < User(2) < Manager(3)`
- ACCESS_LEVEL 比较：`Owner(3) > Admin(2) > Manager(1) > User(0)`

**关键实现**：[i32 impl PartialOrd<MembershipType>](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/organization.rs#L169-L184)
- 将 i32 转换为 MembershipType 后使用 ACCESS_LEVEL 比较
- 所以 `m.atype < MembershipType::Admin` 实际比较的是 ACCESS_LEVEL

```rust
// 角色豁免判断逻辑（策略不约束 Owner 和 Admin）
if m.atype < MembershipType::Admin {
    // Manager 和 User 才会执行策略检查
}
```

### 2.3 策略适用范围判定函数

#### is_applicable_to_user

核心判定函数：[OrgPolicy::is_applicable_to_user](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L260-L281)

**判定逻辑：**
1. **策略查询**：调用 `find_accepted_and_confirmed_by_user_and_active_policy`
   - 用户状态：Accepted 或 Confirmed
   - 策略状态：已启用
   
2. **角色检查**：调用 `find_confirmed_by_user_and_org`
   - 用户状态：**仅 Confirmed**
   - 角色：`user.atype < MembershipType::Admin`（即 User/Manager）

⚠️ **注意不一致性**：策略查询包含 Accepted 用户，但角色检查仅针对 Confirmed 用户。

#### find_accepted_and_confirmed_by_user_and_active_policy

[org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L212-L233)

**过滤条件：**
- 用户状态：Accepted OR Confirmed
- 策略类型：匹配指定类型
- 策略状态：enabled = true

#### find_confirmed_by_user_and_active_policy

[org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L235-L255)

**过滤条件：**
- 用户状态：仅 Confirmed
- 策略类型：匹配指定类型
- 策略状态：enabled = true

### 2.4 check_user_allowed 统一入口

[OrgPolicy::check_user_allowed](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L283-L317)

**前置条件：**
```rust
if m.atype < MembershipType::Admin && m.status > MembershipStatus::Invited as i32 {
    // 仅对非 Admin 且状态 > Invited（Accepted/Confirmed）的用户执行策略检查
}
```

**策略检查：**
1. TwoFactorAuthentication 策略
2. SingleOrg 跨组织检查
3. SingleOrg 本组织检查

**调用位置：**
- 确认邀请：[confirm_invite_impl](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L1427)
- 恢复成员：[restore_member_impl](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2402)
- 编辑成员、激活成员

## 3. 各策略执行点分析

### 3.1 双因素认证策略 (TwoFactorAuthentication)

**执行位置：** [OrgPolicy::check_user_allowed](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L285-L295)

**触发时机：**
- 确认用户加入组织时
- 恢复用户成员资格时
- 编辑用户成员信息时
- 激活成员时

**状态约束：** 用户状态 > Invited（Accepted/Confirmed）

**执行逻辑：**
```rust
if policy.enabled && TwoFactor::find_by_user(user_uuid).await.is_empty() {
    if CONFIG.email_2fa_auto_fallback() {
        // 自动激活邮件2FA
        two_factor::email::find_and_activate_email_2fa(...)
    } else {
        err!("Cannot {action} because 2FA is required")
    }
}
```

**策略启用后的存量影响：**
- [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2073-L2082)
- 调用 `enforce_2fa_policy_for_org` 遍历组织所有成员
- **立即撤销**未启用 2FA 的非 Admin 成员（状态设置为 Revoked）
- 记录 OrganizationUserRevoked 事件

**策略禁用后的影响：**
- 后续加入的用户不再强制 2FA
- 已被撤销的用户**不会自动恢复**，需手动重新邀请或恢复

### 3.2 单一组织策略 (SingleOrg)

**执行位置1：** [OrgPolicy::check_user_allowed](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L297-L313)

**执行位置2：** 创建组织时 [create_organization](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L198-L202)

**触发时机：**
- 创建新组织时（检查用户是否属于启用 SingleOrg 的组织）
- 确认/恢复用户成员资格时
- 用户加入其他组织时

**状态约束：** 用户状态 > Invited（Accepted/Confirmed）

**双向检查逻辑：**
1. **跨组织检查**：用户是否属于其他已启用 SingleOrg 策略的组织
   - 使用 `is_applicable_to_user`，排除当前组织

2. **本组织检查**：本组织启用 SingleOrg 时，用户是否已加入其他组织
   - 调用 `count_accepted_and_confirmed_by_user` 检查用户在其他组织的成员数

**策略启用后的存量影响：**
- [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2085-L2117)
- 遍历组织所有成员
- **立即撤销**非 Admin 且已加入其他组织的用户
- 发送邮件通知（如配置）
- 记录 OrganizationUserRemoved 事件

**策略禁用后的影响：**
- 后续用户可加入多个组织
- 已被撤销的用户**不会自动恢复**

### 3.3 个人所有权策略 (PersonalOwnership)

**执行位置：** [enforce_personal_ownership_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs#L384-L393)

**触发时机（仅密码条目相关，不涉及 Send）：**
- [分享密码时](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs#L339)
- [创建/更新密码条目时](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs#L418) ([update_cipher_from_data])
- [导入密码时](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs#L597)

**⚠️ 重要校正：** Send 路径不执行 PersonalOwnership 策略检查。Send 的创建/更新仅检查 `DisableSend` 和 `SendOptions` 策略。

**状态约束：** 使用 `is_applicable_to_user`
- 策略查询：Accepted/Confirmed 用户
- 角色检查：Confirmed 用户且角色 < Admin

**执行逻辑：**
```rust
if cipher.belongs_to_personal_vault() {
    if OrgPolicy::is_applicable_to_user(user_id, PersonalOwnership, None, conn).await {
        err!("Due to an Enterprise Policy, you are restricted from saving items to your personal vault.")
    }
}
```

**策略启用后的存量影响：**
- 用户无法创建新的个人密码条目
- 用户无法更新已有的个人密码条目
- 用户仍然可以：删除个人条目、将个人条目分享到组织
- **存量个人条目不会被自动删除或移动**

**策略禁用后的影响：**
- 用户恢复创建/更新个人条目的能力
- 无存量数据变更

### 3.4 禁用 Send 策略 (DisableSend)

**执行位置：** [enforce_disable_send_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/sends.rs#L101-L108)

**触发时机：**
- 创建 Send 时
- 更新 Send 时

**状态约束：** 使用 `is_applicable_to_user`
- 策略查询：Accepted/Confirmed 用户
- 角色检查：Confirmed 用户且角色 < Admin

**执行逻辑：**
```rust
if !CONFIG.sends_allowed() 
    || OrgPolicy::is_applicable_to_user(user_id, DisableSend, None, conn).await 
{
    err!("Due to an Enterprise Policy, you are only able to delete an existing Send.")
}
```

**全局覆盖：** 配置项 `sends_allowed` 可以全局禁用 Send 功能，优先级高于组织策略。

**策略启用后的存量影响：**
- 用户无法创建新的 Send
- 用户无法更新已有的 Send
- 用户仍然可以删除 Send
- **存量 Send 不会被自动删除**

**策略禁用后的影响：**
- 用户恢复创建/更新 Send 的能力
- 无存量数据变更

### 3.5 Send 选项策略 (SendOptions)

**执行位置：** [enforce_disable_hide_email_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/sends.rs#L117-L125)

**子选项：**
- `disable_hide_email`: 禁用隐藏邮件功能

**状态约束：** 使用 `find_confirmed_by_user_and_active_policy` + 角色检查
- 仅 Confirmed 用户
- 角色 < Admin

**执行逻辑：**
```rust
if hide_email && OrgPolicy::is_hide_email_disabled(user_id, conn).await {
    err!("Due to an Enterprise Policy, you are not allowed to hide your email address...")
}
```

**策略启用后的存量影响：**
- 用户创建/更新 Send 时无法设置 hide_email = true
- **存量 Send 的 hide_email 设置不会被自动修改**

**策略禁用后的影响：**
- 用户恢复设置 hide_email 的能力
- 无存量数据变更

### 3.6 主密码复杂度策略 (MasterPassword)

**执行位置：** [master_password_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/mod.rs#L93-L125)

**服务端处理：** ⚠️ 仅合并后返回客户端，无强制执行

**策略合并逻辑：**
- 收集用户所属所有组织的已启用主密码策略
- 调用 `find_accepted_and_confirmed_by_user_and_active_policy`（Accepted/Confirmed 用户）
- 合并规则：取最严格的值
  - 数值型：取最大值（min_complexity, min_length）
  - 布尔型：取逻辑或（require_lower, require_upper, require_numbers, require_special, enforce_on_login）

**返回时机：**
- 在 [identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/identity.rs#L502-L548) 登录响应中返回给客户端
- 客户端在用户修改主密码时本地强制执行复杂度要求

**配置覆盖：** SSO 启用时，全局配置 `sso_master_password_policy` 可作为兜底策略。

**⚠️ 重要说明：**
- 服务端没有任何校验主密码复杂度的执行点
- 完全依赖客户端在修改密码时执行检查
- 用户可通过修改客户端或直接调用 API 绕过策略
- 属于"服务端存储 + 合并 + 返回客户端"模式，无服务端强制执行

**策略启用后的存量影响：**
- 客户端会收到合并后的策略配置
- 客户端在用户修改主密码时强制执行复杂度要求
- **服务端不强制校验现有主密码**
- 存量用户的主密码不会被自动检查或重置

**策略禁用后的影响：**
- 后续主密码修改不再受该组织策略约束
- 无存量数据变更

### 3.7 密码重置策略 (ResetPassword)

**策略数据结构**：[ResetPasswordDataModel](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L62-L68)
```rust
pub struct ResetPasswordDataModel {
    #[serde(rename = "autoEnrollEnabled", alias = "AutoEnrollEnabled")]
    pub auto_enroll_enabled: bool,  // 是否自动登记密码重置密钥
}
```

---

#### 核心校验函数：check_reset_password_applicable

所有重置密码相关 API 的统一入口校验：[check_reset_password_applicable](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L3027-L3041)

```rust
async fn check_reset_password_applicable(org_id: &OrganizationId, conn: &DbConn) -> EmptyResult {
    // 第一步：邮件开关校验
    if !CONFIG.mail_enabled() {
        err!("Password reset is not supported on an email-disabled instance.");
    }

    // 第二步：策略存在校验
    let Some(policy) = OrgPolicy::find_by_org_and_type(org_id, OrgPolicyType::ResetPassword, conn).await else {
        err!("Policy not found")
    };

    // 第三步：策略启用校验
    if !policy.enabled {
        err!("Reset password policy not enabled");
    }

    Ok(())
}
```

**三层校验顺序**：
1. **全局邮件开关** → 邮件禁用直接拒绝
2. **策略存在性** → 策略未创建直接拒绝
3. **策略启用状态** → 策略未启用直接拒绝

⚠️ **重要**：邮件配置是 ResetPassword 策略的前置依赖，邮件禁用时即使策略启用也无法使用。

---

#### 执行点 1：新用户接受邀请时强制提供密钥

**执行位置**：[accept_invite](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L1307-L1311)

**策略检查函数**：[org_is_reset_password_auto_enroll](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L319-L331)
```rust
pub async fn org_is_reset_password_auto_enroll(org_uuid: &OrganizationId, conn: &DbConn) -> bool {
    match OrgPolicy::find_by_org_and_type(org_uuid, OrgPolicyType::ResetPassword, conn).await {
        Some(policy) => match serde_json::from_str::<ResetPasswordDataModel>(&policy.data) {
            Ok(opts) => {
                // 必须同时满足：策略启用 + auto_enroll_enabled = true
                return policy.enabled && opts.auto_enroll_enabled;
            }
            _ => error!("Failed to deserialize ResetPasswordDataModel: {}", policy.data),
        },
        None => return false,
    }
    false
}
```

**接受邀请时的执行逻辑**：
```rust
let reset_password_key = match OrgPolicy::org_is_reset_password_auto_enroll(&membership.org_uuid, &conn).await {
    // autoEnroll 启用且未提供密钥 → 拒绝
    true if data.reset_password_key.is_none() => err!("Reset password key is required, but not provided."),
    // autoEnroll 启用且提供了密钥 → 使用该密钥
    true => data.reset_password_key,
    // autoEnroll 未启用 → 不强制
    false => None,
};
```

**触发条件**：
- 策略启用且 `auto_enroll_enabled = true`
- 用户接受组织邀请时

---

#### 执行点 2：成员登记/撤销恢复密钥

**执行位置**：[put_reset_password_enrollment](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L3043-L3093)

**API 端点**：`PUT /organizations/<org_id>/users/<user_id>/reset-password-enrollment`

**完整执行流程**：

```
用户发起登记/撤销请求
    │
    ▼
1. 身份校验：user_id == headers.user.uuid
   │ 只能操作自己的密钥
   │
    ▼
2. check_reset_password_applicable
   ├─► 邮件开关校验
   ├─► 策略存在校验
   └─► 策略启用校验
   │
    ▼
3. 密钥标准化处理
   │
   ├─► None → 表示撤销
   ├─► Some("") 空字符串 → 转换为 None（撤销）
   └─► Some(非空密钥) → 表示登记
   │
    ▼
4. ⚠️ autoEnroll 撤销拦截检查
   │ if (撤销操作) && (auto_enroll_enabled = true)
   │    → err!("Reset password can't be withdrawn due to an enterprise policy")
   │
    ▼
5. 登记时的额外校验
   │ if (登记操作)
   │    ├─► 验证主密码哈希
   │    └─► 验证 OTP（如启用）
   │
    ▼
6. 更新数据库
   │ membership.reset_password_key = reset_password_key
   │
    ▼
7. 记录事件
   ├─► 登记 → OrganizationUserResetPasswordEnroll
   └─► 撤销 → OrganizationUserResetPasswordWithdraw
```

**autoEnroll 为什么能挡住撤销？**

在 [第 3067-3069 行](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L3067-L3069)：
```rust
// 如果用户请求撤销（reset_password_key = None）
// 同时策略配置了 auto_enroll_enabled = true
if reset_password_key.is_none() && OrgPolicy::org_is_reset_password_auto_enroll(&org_id, &conn).await {
    // 直接拒绝撤销请求
    err!("Reset password can't be withdrawn due to an enterprise policy");
}
```

**设计意图**：当组织启用自动登记模式时，管理员希望所有成员都有密码重置密钥，因此禁止用户主动撤销。

---

#### 执行点 3：管理员重置成员主密码

**执行位置**：[put_reset_password](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2908-L2968)

**API 端点**：`PUT /organizations/<org_id>/users/<member_id>/reset-password`

**权限要求**：AdminHeaders（管理员或所有者）

**完整执行流程**：

```
管理员发起重置请求
    │
    ▼
1. 组织匹配校验：org_id == headers.org_id
   │
    ▼
2. 组织存在性校验
   │
    ▼
3. 成员存在性校验（成员属于该组织）
   │
    ▼
4. 用户存在性校验
   │
    ▼
5. check_reset_password_applicable_and_permissions
   ├─► check_reset_password_applicable
   │    ├─► 邮件开关
   │    ├─► 策略存在
   │    └─► 策略启用
   │
   └─► 角色权限校验 [L3019-L3024]
        ├─► Owner → 通过（可重置任何人）
        ├─► Admin 且目标角色 <= Admin → 通过
        └─► 其他情况 → 拒绝
   │
    ▼
6. 成员状态校验
   ├─► 必须已登记密钥（reset_password_key.is_some()）
   └─► 状态必须为 Confirmed
   │
    ▼
7. 发送重置通知邮件 [send_admin_reset_password]
   │ （先发送邮件，确保配置正确再执行重置）
   │
    ▼
8. 更新用户密码
   │ user.set_password(new_master_password_hash, key, ...)
   │
    ▼
9. 强制所有设备登出
   │ nt.send_logout(&user, None, &conn)
   │
    ▼
10. 记录事件：OrganizationUserAdminResetPassword
```

**角色权限校验逻辑**：[L3019-L3024](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L3019-L3024)
```rust
match headers.membership_type {
    // Owner 可以重置任何人
    MembershipType::Owner => Ok(()),
    // Admin 只能重置角色级别 <= Admin 的成员
    // 即：User(2)、Manager(3)、Admin(1)，但不能重置 Owner(0)
    MembershipType::Admin if target_user.atype <= MembershipType::Admin => Ok(()),
    _ => err!("No permission to reset this user's password"),
}
```

---

#### 执行点 4：管理员获取重置密码详情

**执行位置**：[get_reset_password_details](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2970-L3005)

**API 端点**：`GET /organizations/<org_id>/users/<member_id>/reset-password-details`

**完整执行流程**：

```
管理员获取详情请求
    │
    ▼
1. 组织匹配校验
   │
    ▼
2. 组织存在性校验
   │
    ▼
3. 成员存在性校验
   │
    ▼
4. 用户存在性校验
   │
    ▼
5. check_reset_password_applicable_and_permissions
   ├─► 邮件开关、策略存在、策略启用
   └─► 角色权限校验（同上）
   │
    ▼
6. 返回重置所需数据
   ├─► kdf: 用户的 KDF 算法类型
   ├─► kdfIterations: 迭代次数
   ├─► kdfMemory: 内存参数（Argon2）
   ├─► kdfParallelism: 并行参数（Argon2）
   ├─► resetPasswordKey: 成员的重置密钥
   └─► encryptedPrivateKey: 组织加密私钥
```

**返回数据用途**：
- KDF 参数：管理员客户端用相同参数派生出新密码哈希
- resetPasswordKey：验证成员已登记
- encryptedPrivateKey：用于加密组织共享密钥

---

#### 策略间依赖约束

**执行位置**：[put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2045-L2070)

当 `enforce_single_org_with_reset_pw_policy` 配置启用时：

```
启用 ResetPassword 策略
    │
    ▼
检查 SingleOrg 是否已启用？
    │
    ├─► 未启用 → err!("Single Organization policy must be enabled first")
    └─► 已启用 → 继续
```

```
禁用 SingleOrg 策略
    │
    ▼
检查 ResetPassword 是否已启用？
    │
    ├─► 已启用 → err!("Reset Password policy must be disabled first")
    └─► 未启用 → 继续
```

**设计原因**：密码重置功能设计上假定用户只属于一个组织，避免跨组织的安全风险。

---

#### 策略启用后的存量影响

| 影响对象 | 具体影响 |
|---------|---------|
| **新加入用户** | 接受邀请时必须提供 reset_password_key（autoEnroll=true时） |
| **存量已登记用户** | 可正常使用密码重置功能，无影响 |
| **存量未登记用户** | 若 auto_enroll_enabled = true：<br>• 无法撤销（但之前也没登记）<br>• 用户可主动登记密钥<br>• 管理员无法重置该用户密码 |
| **管理员操作** | 可对已登记密钥的成员执行密码重置 |

**注意**：
- 策略启用后**不会自动强制**存量用户登记密钥
- 仅影响新加入的用户和后续的登记/撤销操作
- 未登记密钥的存量用户，管理员无法重置其密码

---

#### 策略禁用后的影响

| 影响对象 | 具体影响 |
|---------|---------|
| **新加入用户** | 接受邀请时无需提供 reset_password_key |
| **存量已登记用户** | reset_password_key **保留在数据库中**<br>但管理员无法再执行密码重置 |
| **存量未登记用户** | 不再受 autoEnroll 约束，可自由选择是否登记 |
| **管理员操作** | 密码重置功能完全不可用 |
| **登记/撤销操作** | 用户无法再登记或撤销密钥 |

**注意**：
- 已设置的 reset_password_key **不会被自动清除**
- 但因策略禁用，所有重置密码相关 API 都会在 `check_reset_password_applicable` 阶段返回错误

### 3.8 密码生成器策略 (PasswordGenerator)

**服务端处理：** ❌ 无强制执行逻辑

**实现说明：**
- 仅在 `OrgPolicyType` 枚举中定义类型值 = 2
- 服务端可通过 API 存储和读取策略配置
- 服务端没有任何执行点或校验逻辑
- 实际强制执行由客户端（浏览器/移动端）完成

**策略启用后的存量影响：**
- 客户端会收到策略配置
- 客户端在密码生成器中应用限制规则
- 服务端不校验生成的密码是否符合策略

**风险提示：** 完全依赖客户端执行，用户可通过修改客户端绕过。

### 3.9 移除 PIN 解锁策略 (RemoveUnlockWithPin)

**服务端处理：** ❌ 无强制执行逻辑

**实现说明：**
- 仅在 `OrgPolicyType` 枚举中定义类型值 = 14
- 服务端可通过 API 存储和读取策略配置
- 服务端没有任何执行点或校验逻辑
- PIN 解锁逻辑完全在客户端本地实现

**策略启用后的存量影响：**
- 客户端会收到策略配置
- 客户端应禁用 PIN 解锁选项
- 服务端无法验证客户端是否真的禁用了 PIN 解锁

### 3.10 限制项目类型策略 (RestrictedItemTypes)

**服务端处理：** ❌ 无强制执行逻辑

**实现说明：**
- 仅在 `OrgPolicyType` 枚举中定义类型值 = 15
- 服务端可通过 API 存储和读取策略配置
- 服务端没有任何执行点或校验逻辑
- 创建/更新密码条目时不检查条目类型

**策略启用后的存量影响：**
- 客户端会收到策略配置
- 客户端在创建条目时应限制可选类型
- 服务端接受任何类型的条目，用户可通过 API 绕过限制

### 3.11 URI 匹配默认值策略 (UriMatchDefaults)

**服务端处理：** ❌ 无强制执行逻辑

**实现说明：**
- 仅在 `OrgPolicyType` 枚举中定义类型值 = 16
- 服务端可通过 API 存储和读取策略配置
- 服务端没有任何执行点或校验逻辑
- URI 匹配逻辑完全在客户端实现

**策略启用后的存量影响：**
- 客户端会收到策略配置
- 客户端在添加 URI 时使用默认匹配规则
- 服务端不校验 URI 的匹配设置

## 4. 策略变更影响汇总

### 4.1 策略启用时的存量动作

| 策略类型 | 启用时是否立即执行存量动作 | 具体动作 |
|---------|--------------------------|---------|
| TwoFactorAuthentication | ✅ 是 | 撤销未启用2FA的非Admin成员 |
| SingleOrg | ✅ 是 | 撤销属于多组织的非Admin成员 |
| PersonalOwnership | ❌ 否 | 仅阻止新操作，存量条目保留 |
| DisableSend | ❌ 否 | 仅阻止新操作，存量Send保留 |
| SendOptions | ❌ 否 | 仅阻止新设置，存量配置保留 |
| MasterPassword | ❌ 否 | 仅返回客户端，无服务端强制执行 |
| ResetPassword | ❌ 否 | 仅影响新用户接受邀请 |
| PasswordGenerator | ❌ 否 | 仅客户端策略 |
| RemoveUnlockWithPin | ❌ 否 | 仅客户端策略 |
| RestrictedItemTypes | ❌ 否 | 仅客户端策略 |
| UriMatchDefaults | ❌ 否 | 仅客户端策略 |

### 4.2 策略禁用后的影响

| 策略类型 | 是否自动恢复存量用户/数据 | 说明 |
|---------|--------------------------|------|
| TwoFactorAuthentication | ❌ 否 | 已撤销用户需手动恢复 |
| SingleOrg | ❌ 否 | 已撤销用户需手动恢复 |
| PersonalOwnership | ✅ 是 | 恢复创建/更新个人条目能力 |
| DisableSend | ✅ 是 | 恢复创建/更新Send能力 |
| SendOptions | ✅ 是 | 恢复hide_email设置能力 |
| MasterPassword | ✅ 是 | 不再强制密码复杂度 |
| ResetPassword | ⚠️ 部分 | 新用户无需reset_password_key<br>但已登记的密钥保留，所有重置功能不可用 |
| 所有客户端策略 | ✅ 是 | 客户端恢复默认行为 |

### 4.3 策略启用流程

在 [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2024-L2140) 中处理策略启用：

```
策略启用
├── 前置校验
│   ├── 策略类型有效性检查
│   └── 策略间依赖检查（如 ResetPassword → SingleOrg）
├── 立即执行动作（仅部分策略）
│   ├── TwoFactorAuthentication: 撤销无2FA的成员
│   └── SingleOrg: 撤销属于多组织的成员
├── 持久化
│   ├── 保存策略到数据库
│   └── 记录 PolicyUpdated 事件
└── 后续影响
    └── 新的 API 请求将受新策略约束
```

### 4.4 策略数据格式

策略的 `data` 字段存储 JSON 格式的配置：

| 策略类型 | data 格式示例 |
|---------|--------------|
| SendOptions | `{"disableHideEmail": true/false}` |
| ResetPassword | `{"autoEnrollEnabled": true/false}` |
| MasterPassword | `{"minLength": 8, "minComplexity": 3, ...}` |
| PasswordGenerator | 客户端定义格式 |
| 其他策略 | 客户端定义格式或 null |

## 5. 策略执行流程图

### 5.1 服务端强制策略执行流程

```
用户发起 API 请求
    │
    ▼
路由处理函数
    │
    ├─► 身份验证与授权（Headers 提取）
    │
    ├─► 策略执行点触发
    │    │
    │    ├─► 状态检查（Accepted/Confirmed？）
    │    │
    │    ├─► 角色检查（豁免 Owner/Admin）
    │    │    └─► m.atype < MembershipType::Admin
    │    │
    │    ├─► 组织策略查询
    │    │    ├─► find_accepted_and_confirmed_by_user_and_active_policy
    │    │    └─► 或 find_confirmed_by_user_and_active_policy
    │    │
    │    └─► 适用判定
    │         └─► is_applicable_to_user
    │
    ├─► 策略适用？ ──否──► 继续执行业务逻辑
    │    │
    │    是
    │    │
    │    ▼
    └─► 执行策略动作
         ├─► 拒绝请求（返回错误）
         ├─► 自动修正（如自动激活邮件2FA）
         └─► 撤销用户成员资格
```

### 5.2 客户端仅策略流程

```
用户发起 API 请求
    │
    ▼
路由处理函数
    │
    ├─► 身份验证与授权
    │
    ├─► 执行业务逻辑
    │    │
    │    └─► ⚠️ 服务端不校验策略
    │
    ▼
客户端
    │
    └─► 本地应用策略限制（如禁用PIN、限制类型）
```

## 6. 关键代码文件索引

| 文件 | 说明 |
|------|------|
| [src/db/models/org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs) | 策略数据模型与核心判定逻辑 |
| [src/db/models/organization.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/organization.rs) | 成员状态、角色类型定义与比较逻辑 |
| [src/api/core/organizations.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs) | 策略 API 端点与启用逻辑 |
| [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs) | PersonalOwnership 策略执行 |
| [src/api/core/sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/sends.rs) | DisableSend/SendOptions 策略执行 |
| [src/api/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/mod.rs) | MasterPassword 策略合并逻辑 |
| [src/api/core/two_factor/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/two_factor/mod.rs) | 2FA 策略强制执行 |
| [src/mail.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/mail.rs) | 管理员重置密码邮件发送 |

## 7. 设计特点总结

### 7.1 角色豁免机制
- Owner 和 Admin 始终豁免策略约束，确保组织管理不被策略锁定
- 通过 ACCESS_LEVEL 比较实现：`m.atype < MembershipType::Admin`

### 7.2 状态过滤不一致性
- 不同策略使用不同的状态过滤条件
- `find_accepted_and_confirmed_*`: 包含 Accepted 和 Confirmed 用户
- `find_confirmed_*`: 仅包含 Confirmed 用户
- `check_user_allowed`: 包含 Accepted 和 Confirmed 用户（`status > Invited`）

### 7.3 策略组合支持
- 用户可属于多个组织，策略按组织独立判定
- MasterPassword 策略支持多组织合并，取最严格值

### 7.4 客户端与服务端策略分离
- **服务端强制策略**：6 种（TwoFactor, SingleOrg, PersonalOwnership, DisableSend, SendOptions, ResetPassword）
- **服务端存储+返回策略**：1 种（MasterPassword，服务端合并后返回客户端，无强制执行）
- **客户端仅策略**：4 种（PasswordGenerator, RemoveUnlockWithPin, RestrictedItemTypes, UriMatchDefaults）
- 客户端策略和 MasterPassword 完全依赖客户端执行，存在绕过风险

### 7.5 配置优先级
- 全局配置可覆盖组织策略（如 `sends_allowed`）
- SSO 配置可作为 MasterPassword 策略兜底

### 7.6 启用即执行 vs 宽松禁用
- **启用严格**：TwoFactor 和 SingleOrg 策略启用时立即对存量用户执行清理动作
- **禁用宽松**：所有策略禁用时不自动回滚已执行的动作，需手动处理

### 7.7 ResetPassword 策略的特殊设计
- **全局依赖**：所有重置密码功能依赖邮件配置 `CONFIG.mail_enabled()`，邮件禁用时策略完全不可用
- **策略间依赖**：可选配置 `enforce_single_org_with_reset_pw_policy` 强制 ResetPassword 依赖 SingleOrg
- **角色权限分层**：Owner 可重置所有人，Admin 只能重置角色 <= Admin 的成员
- **autoEnroll 锁定**：自动登记模式下用户无法主动撤销密钥
- **两步校验**：登记密钥时需要验证主密码和 OTP
