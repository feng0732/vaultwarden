# Vaultwarden 组织策略执行机制分析

## 1. 策略数据模型

### 1.1 策略类型定义

策略类型定义在 [org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L31-L52) 的 `OrgPolicyType` 枚举中：

| 策略类型 | 值 | 说明 |
|---------|----|------|
| TwoFactorAuthentication | 0 | 双因素认证强制策略 |
| MasterPassword | 1 | 主密码复杂度策略 |
| PasswordGenerator | 2 | 密码生成器策略 |
| SingleOrg | 3 | 单一组织限制策略 |
| PersonalOwnership | 5 | 个人所有权限制策略 |
| DisableSend | 6 | 禁用 Send 功能策略 |
| SendOptions | 7 | Send 选项策略 |
| ResetPassword | 8 | 密码重置策略 |
| RemoveUnlockWithPin | 14 | 移除 PIN 解锁策略 |
| RestrictedItemTypes | 15 | 限制项目类型策略 |
| UriMatchDefaults | 16 | URI 匹配默认值策略 |

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

### 2.1 策略适用范围判定

策略的核心判定函数是 [OrgPolicy::is_applicable_to_user](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L260-L281)：

**判定逻辑：**
1. 用户属于已启用该策略的组织
2. 用户在该组织中的角色 **不是 Owner 或 Admin** (`user.atype < MembershipType::Admin`)
3. 用户状态为 Accepted 或 Confirmed

### 2.2 角色层级约束

角色类型定义在组织模型中，权限从高到低：
- **Owner (所有者)**: 不受策略约束
- **Admin (管理员)**: 不受策略约束  
- **Manager (经理)**: 受策略约束
- **User (普通用户)**: 受策略约束

**角色豁免规则：** 所有策略的 `is_applicable_to_user` 检查都会豁免 Owner 和 Admin 角色。

## 3. 各策略执行点分析

### 3.1 双因素认证策略 (TwoFactorAuthentication)

**执行位置：** [OrgPolicy::check_user_allowed](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L283-L317)

**触发时机：**
- 确认用户加入组织时 ([confirm_invite_impl](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L1427))
- 恢复用户成员资格时 ([restore_member_impl](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2402))
- 编辑用户成员信息时
- 激活成员时

**执行逻辑：**
```rust
if m.atype < MembershipType::Admin && m.status > Invited {
    if policy.enabled && user_has_no_2fa {
        if CONFIG.email_2fa_auto_fallback() {
            // 自动激活邮件2FA
            two_factor::email::find_and_activate_email_2fa(...)
        } else {
            err!("Cannot {action} because 2FA is required")
        }
    }
}
```

**策略启用后的影响：** [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2073-L2082)
- 调用 `enforce_2fa_policy_for_org` 撤销所有未启用2FA的非管理员成员

### 3.2 单一组织策略 (SingleOrg)

**执行位置：** [OrgPolicy::check_user_allowed](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L297-L313)

**触发时机：**
- 创建新组织时 ([create_organization](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L198-L202))
- 确认/恢复用户成员资格时
- 用户加入其他组织时

**双向检查逻辑：**
1. **跨组织检查**：用户是否属于其他已启用 SingleOrg 策略的组织
2. **本组织检查**：本组织启用 SingleOrg 时，用户是否已加入其他组织

**策略启用后的影响：** [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2085-L2117)
- 遍历组织所有成员
- 对非 Admin 且已加入其他组织的用户执行撤销操作
- 发送邮件通知（如配置）
- 记录 OrganizationUserRemoved 事件

### 3.3 个人所有权策略 (PersonalOwnership)

**执行位置：** [enforce_personal_ownership_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs#L384-L393)

**触发时机：**
- 创建/更新密码条目时 ([update_cipher_from_data](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs#L418))
- 创建/更新 Send 时

**执行逻辑：**
```rust
if cipher.belongs_to_personal_vault() {
    if OrgPolicy::is_applicable_to_user(user_id, PersonalOwnership, None, conn).await {
        err!("Due to an Enterprise Policy, you are restricted from saving items to your personal vault.")
    }
}
```

**注意：** 策略只限制保存到个人保险库，用户仍然可以：
- 删除个人保险库中的已有条目
- 将个人条目分享到组织

### 3.4 禁用 Send 策略 (DisableSend)

**执行位置：** [enforce_disable_send_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/sends.rs#L101-L108)

**触发时机：**
- 创建 Send 时
- 更新 Send 时

**执行逻辑：**
```rust
if !CONFIG.sends_allowed() 
    || OrgPolicy::is_applicable_to_user(user_id, DisableSend, None, conn).await 
{
    err!("Due to an Enterprise Policy, you are only able to delete an existing Send.")
}
```

**全局覆盖：** 配置项 `sends_allowed` 可以全局禁用 Send 功能，优先级高于组织策略。

### 3.5 Send 选项策略 (SendOptions)

**执行位置：** [enforce_disable_hide_email_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/sends.rs#L117-L125)

**子选项：**
- `disable_hide_email`: 禁用隐藏邮件功能

**执行逻辑：**
```rust
if hide_email && OrgPolicy::is_hide_email_disabled(user_id, conn).await {
    err!("Due to an Enterprise Policy, you are not allowed to hide your email address...")
}
```

### 3.6 主密码复杂度策略 (MasterPassword)

**执行位置：** [master_password_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/mod.rs#L93-L125)

**策略合并逻辑：**
- 收集用户所属所有组织的已启用主密码策略
- 合并规则：取最严格的值
  - 数值型：取最大值（min_complexity, min_length）
  - 布尔型：取逻辑或（require_lower, require_upper, require_numbers, require_special, enforce_on_login）

**配置覆盖：** SSO 启用时，全局配置 `sso_master_password_policy` 可作为兜底策略。

### 3.7 密码重置策略 (ResetPassword)

**执行位置：**
- 组织自动注册检查：[OrgPolicy::org_is_reset_password_auto_enroll](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs#L319-L331)
- 接受邀请时检查：[accept_invite](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L1307-L1311)

**策略依赖约束：** [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2045-L2070)
- 当 `enforce_single_org_with_reset_pw_policy` 配置启用时：
  - 启用 ResetPassword 前必须先启用 SingleOrg
  - ResetPassword 启用时不能禁用 SingleOrg

## 4. 策略变更影响分析

### 4.1 策略启用流程

在 [put_policy](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs#L2024-L2140) 中处理策略启用：

```
策略启用
├── 前置校验
│   ├── 策略类型有效性检查
│   └── 策略间依赖检查（如 ResetPassword → SingleOrg）
├── 立即执行动作
│   ├── TwoFactorAuthentication: 撤销无2FA的成员
│   └── SingleOrg: 撤销属于多组织的成员
├── 持久化
│   ├── 保存策略到数据库
│   └── 记录 PolicyUpdated 事件
└── 后续影响
    └── 新的 API 请求将受新策略约束
```

### 4.2 策略禁用的影响

策略禁用后：
- 后续的 API 请求将不再受该策略约束
- **已被策略影响的用户不会自动恢复**（如被 SingleOrg 策略撤销的成员需要手动恢复）
- 策略数据保留在数据库中，`enabled` 标记为 false

### 4.3 策略数据变更的影响

策略的 `data` 字段存储 JSON 格式的配置：
- **SendOptions**: `{"disableHideEmail": true/false}`
- **ResetPassword**: `{"autoEnrollEnabled": true/false}`
- **MasterPassword**: 复杂度配置对象

修改策略配置后，所有后续请求将使用新配置。

## 5. 策略执行流程图

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
    │    ├─► 角色检查（豁免 Owner/Admin）
    │    │
    │    ├─► 组织策略查询
    │    │    └─► OrgPolicy::find_by_org_and_type
    │    │
    │    └─► 适用判定
    │         └─► OrgPolicy::is_applicable_to_user
    │              ├─► 策略是否启用？
    │              ├─► 用户角色 < Admin？
    │              └─► 用户状态为 Accepted/Confirmed？
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

## 6. 关键代码文件索引

| 文件 | 说明 |
|------|------|
| [src/db/models/org_policy.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/db/models/org_policy.rs) | 策略数据模型与核心判定逻辑 |
| [src/api/core/organizations.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/organizations.rs) | 策略 API 端点与启用逻辑 |
| [src/api/core/ciphers.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/ciphers.rs) | PersonalOwnership 策略执行 |
| [src/api/core/sends.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/sends.rs) | DisableSend/SendOptions 策略执行 |
| [src/api/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/mod.rs) | MasterPassword 策略合并逻辑 |
| [src/api/core/two_factor/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/12-vaultwarden/src/api/core/two_factor/mod.rs) | 2FA 策略强制执行 |

## 7. 设计特点总结

1. **角色豁免机制**：Owner 和 Admin 始终豁免策略约束，确保组织管理不被策略锁定
2. **策略组合支持**：用户可属于多个组织，策略按组织独立判定
3. **配置优先级**：全局配置可覆盖组织策略（如 `sends_allowed`）
4. **启用即执行**：部分策略启用时立即对现有用户执行清理动作
5. **宽松禁用**：策略禁用时不自动回滚已执行的动作，需手动处理
6. **多策略合并**：主密码等策略支持多组织策略合并，取最严格值
