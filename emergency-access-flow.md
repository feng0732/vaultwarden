# Vaultwarden 紧急访问流程（代码实现视角）

> 源码版本基于当前代码库，核心文件：
> - 数据模型：[emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/db/models/emergency_access.rs)
> - API 路由：[emergency_access.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs)
> - 数据库 Schema：[up.sql](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/migrations/sqlite/2021-08-30-193501_create_emergency_access/up.sql)
> - 邮件通知：[mail.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/mail.rs)
> - 认证令牌：[auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/auth.rs)
> - 定时任务注册：[main.rs](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/main.rs#L701-L716)

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
* Confirmed 状态后 Grantor 可随时通过 update 接口修改 wait_time_days 和 atype
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

#### ⚠️ 定时任务注释与默认配置的矛盾点

[main.rs#L701-L703](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/main.rs#L701-L703) 的注释明确说明：

> "This job should run **before** the emergency access reminders job to avoid sending reminders for requests that are about to be granted anyway."

**意图**：timeout_job 应该先执行，这样对于已经满足等待期条件的请求，就不会再发送不必要的提醒。

**实际默认配置**（[config.rs#L551-L557](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/config.rs#L551-L557)）：

| 任务 | cron 表达式 | 执行时间 |
|------|-------------|----------|
| `emergency_notification_reminder_schedule` | `0 3 * * * *` | 每小时第 3 分钟 |
| `emergency_request_timeout_schedule` | `0 7 * * * *` | 每小时第 7 分钟 |

**分析**：

- 实际执行顺序与注释意图**相反**：reminder_job（第 3 分钟）先于 timeout_job（第 7 分钟）执行
- **后果**：在等待期刚好到期的那个小时内，reminder_job 会先发送"还剩 1 天"的提醒，4 分钟后 timeout_job 才执行批准操作
- **但影响有限**：因为提醒的触发条件是 `wait_time_days - 1` 天（到期前一天），所以在到期当天 reminder_job 可能会发一次不必要的提醒（除非 `wait_time_days == 1`）

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

## 五、更新机制：等待期与访问类型的动态调整

### 5a. 更新接口

**API**：
- `PUT /emergency-access/<emer_id>`
- `POST /emergency-access/<emer_id>`

两个路由**共用同一逻辑**（见 [put_emergency_access#L122](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L122)），属于 REST API 兼容设计。

### 5b. 更新分支分析

核心更新逻辑（[post_emergency_access#L126-L155](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L126-L155)）：

```rust
emergency_access.atype = new_type;
emergency_access.wait_time_days = data.wait_time_days;
if data.key_encrypted.is_some() {
    emergency_access.key_encrypted = data.key_encrypted;
}
emergency_access.save(&conn).await?;
```

**关键点**：
1. **无状态检查**：接口仅验证请求者是 Grantor，但**不检查当前 status**
2. `atype` 和 `wait_time_days` **无条件覆盖**
3. `key_encrypted` 采用**补写模式**

### 5c. 等待期动态调整的实际影响

由于更新接口**没有状态限制**，`wait_time_days` 可以在**任何状态下**被修改，包括 `RecoveryInitiated`。

**场景分析**（假设原等待期 7 天）：

| 时间点 | 操作 | `recovery_initiated_at` | `wait_time_days` | 到期时间 |
|--------|------|------------------------|-----------------|----------|
| Day 0 10:00 | Grantee initiate 恢复 | Day 0 10:00 | 7 | Day 7 10:00 |
| Day 5 14:00 | Grantor 修改为 14 天 | Day 0 10:00 (不变) | 14 | Day 14 10:00 |
| Day 10 09:00 | Grantor 修改为 3 天 | Day 0 10:00 (不变) | 3 | **立即批准**（Day 3 已过） |

**结论**：
- `recovery_initiated_at` 作为固定起点，`wait_time_days` 作为可变窗口
- **Grantor 可在恢复等待期内延长或缩短等待时间**
- 缩短到已过去的时间会导致下一次 timeout_job 运行时**立即批准**
- 修改不会触发任何邮件通知，受托人不会收到变更提醒

### 5d. 访问类型动态调整

`atype` 同样可以在**任何状态下**修改：

| 状态 | 行为 | 实际影响 |
|------|------|----------|
| `Confirmed` | View ↔ Takeover 可互转 | 影响后续 initiate 后可执行的操作 |
| `RecoveryInitiated` | View ↔ Takeover 可互转 | 动态改变批准后可执行的操作类型 |
| `RecoveryApproved` | View ↔ Takeover 可互转 | 改变后立即影响 `is_valid_request()` 校验 |

⚠️ **重要**：即使已经批准（RecoveryApproved），Grantor 仍可以将 Takeover 改为 View，从而阻止受托人执行密码重置操作。

### 5e. 共享密钥补写条件

`key_encrypted` 字段的更新逻辑：

```rust
if data.key_encrypted.is_some() {
    emergency_access.key_encrypted = data.key_encrypted;
}
```

**条件解读**：
- 只有当请求中**明确提供** `keyEncrypted` 字段（且不为 null）时才更新
- 如果请求中**不包含**该字段，则**保留数据库现有值不变**

**使用场景**：
1. **首次确认**：通过 `confirm` 接口设置初始 `key_encrypted`
2. **后续补写/重写**：通过 `put/post` 接口更新密钥（如 Grantor 轮换主密钥后）
3. **受托人重新注册**：Grantor 可在不重新邀请的情况下更新加密密钥

---

## 六、撤销规则

### 6a. 授权人拒绝恢复（Reject）

**API**: `POST /emergency-access/<emer_id>/reject`

- 由 Grantor 发起
- 前提状态：`RecoveryInitiated(3)` **或** `RecoveryApproved(4)` 均可
- 效果：状态回退到 `Confirmed(2)`（见 [reject_emergency_access#L539](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/api/core/emergency_access.rs#L539)）
- `recovery_initiated_at` **不清空**（仅 status 变化），但下次发起恢复时会重新设置
- 发送 `emergency_access_recovery_rejected` 邮件给受托人
- **注意：即使已经 RecoveryApproved，Grantor 仍然可以 reject，但若受托人已经执行了 Takeover 的 password 步骤则无法挽回**

### 6b. 删除紧急访问关系（Delete）

**API**: `DELETE /emergency-access/<emer_id>`

- Grantor 或 Grantee 均可发起
- 无论当前处于何种状态，均直接从数据库删除记录
- 同时触发 Grantor 的 `update_uuid_revision`（使客户端同步）

### 6c. 授权人主动批准（Approve）

**API**: `POST /emergency-access/<emer_id>/approve`

- 由 Grantor 发起，前提状态必须是 `RecoveryInitiated(3)`
- 效果：状态变为 `RecoveryApproved(4)`，**跳过等待期**
- 发送 `emergency_access_recovery_approved` 邮件给受托人

---

## 七、过期规则

| 场景 | 规则 |
|------|------|
| 邀请 JWT 过期 | 由 `INVITATION_EXPIRATION_HOURS` 控制（默认值取决于配置），过期后 accept 请求会被 JWT 验证拒绝 |
| 等待期超时自动批准 | `recovery_initiated_at + wait_time_days <= now` 时由 `emergency_request_timeout_job` 自动批准 |
| 提醒通知时机 | 等待期到期前 1 天发送最终提醒，之后每 24 小时发送一次 |
| 无整体过期 | 紧急访问关系本身**不会过期**，除非被主动删除 |

---

## 八、全局开关

配置项 `EMERGENCY_ACCESS_ALLOWED`（默认 `true`），定义于 [config.rs#L639](file:///d:/fz/0601/solo-dogfeeding/code/5-vaultwarden/src/config.rs#L639)。

- 关闭后所有 API 端点返回错误 `"Emergency access is not enabled."`
- 定时任务也会在开头检查此开关，关闭时直接 return

---

## 九、数据库字段速查

| 字段 | 类型 | 说明 |
|------|------|------|
| `uuid` | TEXT PK | 记录唯一标识 |
| `grantor_uuid` | TEXT FK→users | 授权人 |
| `grantee_uuid` | TEXT FK→users (nullable) | 受托人（Invited 阶段为空） |
| `email` | TEXT (nullable) | 受托人邮箱（Accepted 后被清空） |
| `key_encrypted` | TEXT (nullable) | 加密密钥（Confirmed 时设置，可后续补写） |
| `atype` | INTEGER | 0=View, 1=Takeover（可随时修改） |
| `status` | INTEGER | 0-4 五种状态 |
| `wait_time_days` | INTEGER | 等待天数（可随时修改，包括等待期内） |
| `recovery_initiated_at` | DATETIME (nullable) | 恢复发起时间（固定起点） |
| `last_notification_at` | DATETIME (nullable) | 上次通知时间（提醒频率控制） |
| `updated_at` | DATETIME | 更新时间 |
| `created_at` | DATETIME | 创建时间 |

---

## 十、邮件通知汇总

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
| **Grantor 修改 wait_time_days/atype** | 无 | 不会发送任何通知 |

---

## 十一、关键行为总结表

| 操作 | 发起者 | 允许的状态 | 状态变化 | 对恢复生效时间的影响 |
|------|--------|-----------|----------|---------------------|
| invite | Grantor | - | → Invited | N/A |
| accept | Grantee | Invited | → Accepted | N/A |
| confirm | Grantor | Accepted | → Confirmed | N/A |
| initiate | Grantee | Confirmed | → RecoveryInitiated | 开始计时：`recovery_initiated_at + wait_time_days` |
| approve | Grantor | RecoveryInitiated | → RecoveryApproved | 立即生效（跳过等待期） |
| reject | Grantor | RecoveryInitiated / RecoveryApproved | → Confirmed | 终止等待，下次 initiate 重新计时 |
| **update atype** | Grantor | 任意 | 不变 | 无，但改变批准后可执行的操作类型 |
| **update wait_time_days** | Grantor | 任意 | 不变 | **动态调整到期时间**（起点不变，窗口改变） |
| delete | Grantor / Grantee | 任意 | 记录删除 | 终止 |
| timeout job | 系统 | RecoveryInitiated | → RecoveryApproved | 等待期到期后自动批准 |
