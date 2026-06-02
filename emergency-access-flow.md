# Vaultwarden 紧急访问流程（代码实现视角）

> 源码版本基于当前代码库，核心文件：
> - 数据模型：[emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/db/models/emergency_access.rs)
> - API 路由：[emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs)
> - 数据库 Schema：[up.sql](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/migrations/sqlite/2021-08-30-193501_create_emergency_access/up.sql)
> - 邮件通知：[mail.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/mail.rs)
> - 认证令牌：[auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/auth.rs)

---

## 一、角色定义

| 角色 | 代码字段 | 说明 |
|------|----------|------|
| **Grantor（授权人）** | `grantor_uuid` | 保险库拥有者，指定紧急联系人 |
| **Grantee（受托人）** | `grantee_uuid` / `email` | 紧急联系人，在授权人生故/无法访问时接管保险库 |

---

## 二、访问类型（EmergencyAccessType）

定义于 [emergency_access.rs#L117-L131](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/db/models/emergency_access.rs#L117-L131)

| 值 | 枚举 | 含义 |
|----|------|------|
| `0` | `View` | 受托人仅可**查看**授权人的密码项目 |
| `1` | `Takeover` | 受托人可**接管**授权人账户，重置主密码 |

---

## 三、状态机（EmergencyAccessStatus）

定义于 [emergency_access.rs#L133-L139](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/db/models/emergency_access.rs#L133-L139)

| 值 | 枚举 | 含义 | 关键特征 |
|----|------|------|----------|
| `0` | `Invited` | 已邀请 | `grantee_uuid` 为空，`email` 有值 |
| `1` | `Accepted` | 已接受 | `grantee_uuid` 已填充，`email` 被清空，等待 Grantor 确认 |
| `2` | `Confirmed` | 已确认 | `key_encrypted` 已设置，关系正式建立 |
| `3` | `RecoveryInitiated` | 恢复已发起 | `recovery_initiated_at` 已记录，等待期开始计时 |
| `4` | `RecoveryApproved` | 恢复已批准 | 受托人可执行 View/Takeover 操作 |

状态流转图：

```
Invited(0) ──accept──▶ Accepted(1) ──confirm──▶ Confirmed(2)
                                                    │
                                          initiate(受托人发起)
                                                    │
                                                    ▼
                                         RecoveryInitiated(3)
                                          │              │
                                   approve(授权人)    等待期超时(定时任务)
                                          │              │
                                          ▼              ▼
                                      RecoveryApproved(4)
                                          │
                               reject(授权人) ──▶ 回到 Confirmed(2)

* 任意状态均可通过 delete 操作彻底删除记录
```

---

## 四、完整流程详解

### 阶段 1：邀请（Invited → Accepted）

**API**: `POST /emergency-access/invite`

1. Grantor 指定受托人邮箱、访问类型（View/Takeover）、等待天数 `wait_time_days`
2. 创建记录，状态为 `Invited(0)`，此时 `grantee_uuid = None`，`email = "grantee@example.com"`
3. 邀请行为取决于邮件和配置：
   - **邮件开启**：发送邀请邮件，邮件中包含 JWT token（含 `emer_id`、`email`、`grantor_name`、`grantor_email`），有效期由 `invitation_expiration_hours` 控制
   - **邮件关闭 + 受托人已是已注册用户**：**自动 accept**，直接跳到 `Accepted(1)` 状态（见 [send_invite#L273-L276](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L273-L276)）
   - **邮件关闭 + 受托人不存在**：创建占位用户并保存 Invitation

**受托人接受邀请**: `POST /emergency-access/<emer_id>/accept`

- 验证 JWT token 中的邮箱与当前登录用户邮箱一致
- 调用 `accept_invite()`：状态变为 `Accepted(1)`，填充 `grantee_uuid`，清空 `email`
- 发送 `emergency_access_invite_accepted` 邮件给 Grantor

### 阶段 2：确认（Accepted → Confirmed）

**API**: `POST /emergency-access/<emer_id>/confirm`

- 由 Grantor 发起，前提状态必须是 `Accepted(1)`
- Grantor 提交加密密钥 `key`（用于加密共享的密钥）
- 状态变为 `Confirmed(2)`，设置 `key_encrypted`，清空 `email`
- 发送 `emergency_access_invite_confirmed` 邮件给受托人
- **此时紧急联系人关系正式建立**，但受托人尚不能访问任何数据

### 阶段 3：发起恢复（Confirmed → RecoveryInitiated）

**API**: `POST /emergency-access/<emer_id>/initiate`

- 由**受托人**发起，前提状态必须是 `Confirmed(2)`
- 设置以下字段：
  - `status = RecoveryInitiated(3)`
  - `recovery_initiated_at = now`（等待期起始时间）
  - `last_notification_at = now`（通知计时起点）
- 发送 `emergency_access_recovery_initiated` 邮件给 Grantor
- **等待期从此刻开始计时**

### 阶段 4：等待期与自动批准

等待期的核心逻辑由两个定时任务实现：

#### 4a. 自动批准（emergency_request_timeout_job）

- **调度**：`EMERGENCY_REQUEST_TIMEOUT_SCHEDULE`，默认 `0 7 * * * *`（每小时第 7 分钟）
- **逻辑**（见 [emergency_access.rs#L722-L775](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L722-L775)）：
  1. 查询所有 `status = RecoveryInitiated(3)` 且 `recovery_initiated_at IS NOT NULL` 的记录
  2. 计算 `recovery_allowed_at = recovery_initiated_at + wait_time_days 天`
  3. 若 `recovery_allowed_at <= now`，则将状态更新为 `RecoveryApproved(4)`
  4. 发送邮件：
     - 给 Grantor：`emergency_access_recovery_timed_out`
     - 给 Grantee：`emergency_access_recovery_approved`

#### 4b. 提醒通知（emergency_notification_reminder_job）

- **调度**：`EMERGENCY_NOTIFICATION_REMINDER_SCHEDULE`，默认 `0 3 * * * *`（每小时第 3 分钟）
- **逻辑**（见 [emergency_access.rs#L777-L834](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L777-L834)）：
  1. 遍历所有 `RecoveryInitiated` 状态的记录
  2. 计算 `final_recovery_reminder_at = recovery_initiated_at + (wait_time_days - 1) 天`（到期前一天）
  3. 若 `final_recovery_reminder_at <= now` 且距上次通知已超过 1 天（或从未通知），则发送提醒
  4. 发送 `emergency_access_recovery_reminder` 邮件给 Grantor

**关键：timeout_job 运行在 reminder_job 之后**（第 7 分钟 vs 第 3 分钟），确保先发提醒再执行超时。主程序注释也明确说明 timeout_job 应先运行（见 [main.rs#L701-L703](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/main.rs#L701-L703)），但实际 cron 配置的执行顺序相反——这是为了避免在同一次调度中先批准再提醒的竞态问题。

### 阶段 5：授权生效后的操作（RecoveryApproved）

受托人访问操作的校验函数见 [is_valid_request()](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L704-L713)：

```rust
fn is_valid_request(emergency_access, requesting_user_id, requested_access_type) -> bool {
    grantee_uuid == requesting_user_id
    && status == RecoveryApproved(4)
    && atype == requested_access_type
}
```

三个条件必须同时满足：请求者是受托人、状态为 RecoveryApproved、访问类型匹配。

#### 5a. View 模式

**API**: `POST /emergency-access/<emer_id>/view`

- 返回 Grantor 的所有 Cipher（密码项目）+ `key_encrypted`
- 受托人**只读**，无法修改

#### 5b. Takeover 模式

**Step 1** — `POST /emergency-access/<emer_id>/takeover`

- 返回 Grantor 的 KDF 参数 + `key_encrypted`
- 受托人需用这些信息准备重置密码

**Step 2** — `POST /emergency-access/<emer_id>/password`

- 受托人提交新的 `new_master_password_hash` 和 `key`
- 系统执行以下操作：
  1. 修改 Grantor 的主密码（[password_emergency_access#L660](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L660)）
  2. 删除 Grantor 的所有两步验证（TwoFactor）（[L664](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L664)）
  3. 将 Grantor 从所有非 Owner 身份的组织中移除（[L667-L671](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L667-L671)）
- **Takeover 完成后 Grantor 可用新密码登录，但原两步验证和组织成员关系已不可逆**

#### 5c. 组织策略查询

**API**: `GET /emergency-access/<emer_id>/policies`

- 受托人在 Takeover 之前可查看 Grantor 所在组织的策略
- 仅 Takeover 类型可用

---

## 五、撤销规则

### 5a. 授权人拒绝恢复（Reject）

**API**: `POST /emergency-access/<emer_id>/reject`

- 由 Grantor 发起
- 前提状态：`RecoveryInitiated(3)` **或** `RecoveryApproved(4)` 均可
- 效果：状态回退到 `Confirmed(2)`（见 [reject_emergency_access#L539](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L539)）
- `recovery_initiated_at` **不清空**（仅 status 变化），但下次发起恢复时会重新设置
- 发送 `emergency_access_recovery_rejected` 邮件给受托人
- **注意：即使已经 RecoveryApproved，Grantor 仍然可以 reject，但若受托人已经执行了 Takeover 的 password 步骤则无法挽回**

### 5b. 删除紧急访问关系（Delete）

**API**: `DELETE /emergency-access/<emer_id>`

- Grantor 或 Grantee 均可发起
- 无论当前处于何种状态，均直接从数据库删除记录
- 同时触发 Grantor 的 `update_uuid_revision`（使客户端同步）

### 5c. 授权人主动批准（Approve）

**API**: `POST /emergency-access/<emer_id>/approve`

- 由 Grantor 发起，前提状态必须是 `RecoveryInitiated(3)`
- 效果：状态变为 `RecoveryApproved(4)`，**跳过等待期**
- 发送 `emergency_access_recovery_approved` 邮件给受托人

---

## 六、过期规则

| 场景 | 规则 |
|------|------|
| 邀请 JWT 过期 | 由 `INVITATION_EXPIRATION_HOURS` 控制（默认值取决于配置），过期后 accept 请求会被 JWT 验证拒绝 |
| 等待期超时自动批准 | `recovery_initiated_at + wait_time_days <= now` 时由 `emergency_request_timeout_job` 自动批准 |
| 提醒通知时机 | 等待期到期前 1 天发送最终提醒，之后每 24 小时发送一次 |
| 无整体过期 | 紧急访问关系本身**不会过期**，除非被主动删除 |

---

## 七、全局开关

配置项 `EMERGENCY_ACCESS_ALLOWED`（默认 `true`），定义于 [config.rs#L639](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/config.rs#L639)。

- 关闭后所有 API 端点返回错误 `"Emergency access is not enabled."`
- 定时任务也会在开头检查此开关，关闭时直接 return

---

## 八、数据库字段速查

| 字段 | 类型 | 说明 |
|------|------|------|
| `uuid` | TEXT PK | 记录唯一标识 |
| `grantor_uuid` | TEXT FK→users | 授权人 |
| `grantee_uuid` | TEXT FK→users (nullable) | 受托人（Invited 阶段为空） |
| `email` | TEXT (nullable) | 受托人邮箱（Accepted 后被清空） |
| `key_encrypted` | TEXT (nullable) | 加密密钥（Confirmed 时设置） |
| `atype` | INTEGER | 0=View, 1=Takeover |
| `status` | INTEGER | 0-4 五种状态 |
| `wait_time_days` | INTEGER | 等待天数 |
| `recovery_initiated_at` | DATETIME (nullable) | 恢复发起时间（等待期起点） |
| `last_notification_at` | DATETIME (nullable) | 上次通知时间（提醒频率控制） |
| `updated_at` | DATETIME | 更新时间 |
| `created_at` | DATETIME | 创建时间 |

---

## 九、邮件通知汇总

| 触发时机 | 收件人 | 邮件模板 |
|----------|--------|----------|
| 邀请发送 | Grantee | `send_emergency_access_invite` |
| 受托人接受 | Grantor | `emergency_access_invite_accepted` |
| Grantor 确认 | Grantee | `emergency_access_invite_confirmed` |
| 受托人发起恢复 | Grantor | `emergency_access_recovery_initiated` |
| Grantor 主动批准 | Grantee | `emergency_access_recovery_approved` |
| 等待期超时自动批准 | Grantor | `emergency_access_recovery_timed_out` |
| 等待期超时自动批准 | Grantee | `emergency_access_recovery_approved` |
| Grantor 拒绝 | Grantee | `emergency_access_recovery_rejected` |
| 等待期到期前提醒 | Grantor | `emergency_access_recovery_reminder` |
