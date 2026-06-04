# Vaultwarden 后台维护任务（Background Maintenance Jobs）

Vaultwarden 有一套基于 cron 的后台定时任务系统，负责过期数据清理、通知推送和状态流转。
整个机制分布在调度层（`main.rs`）、配置层（`config.rs`）和业务层（各 model / api 文件）三处，协作关系如下。

---

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────┐
│  schedule_jobs()                  main.rs:663            │
│  ┌───────────────────────────────────────────────────┐  │
│  │  独立 OS 线程 "job-scheduler"                      │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  tokio::Runtime (专用)                       │  │  │
│  │  │  JobScheduler (job_scheduler_ng)             │  │  │
│  │  │    ├─ Job: send_purge                       │  │  │
│  │  │    ├─ Job: trash_purge                      │  │  │
│  │  │    ├─ Job: incomplete_2fa                   │  │  │
│  │  │    ├─ Job: emergency_request_timeout        │  │  │
│  │  │    ├─ Job: emergency_notification_reminder  │  │  │
│  │  │    ├─ Job: auth_request_purge               │  │  │
│  │  │    ├─ Job: duo_context_purge                │  │  │
│  │  │    ├─ Job: event_cleanup                    │  │  │
│  │  │    └─ Job: sso_auth_purge                   │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │  loop { sched.tick(); sleep(poll_interval) }       │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**关键入口**：[schedule_jobs](src/main.rs#L663) 在服务器启动时被调用，在独立的 OS 线程中创建 `tokio::Runtime` 和 `JobScheduler`，
通过 `loop { sched.tick(); sleep }` 循环驱动。

---

## 2. 定时触发机制

### 2.1 调度器：job_scheduler_ng

调度器使用 `job_scheduler_ng` crate（[Cargo.toml:140](Cargo.toml#L140)），它是一个纯内存的 cron 调度器，
不依赖外部存储。核心调用链：

```rust
// main.rs:757-759
loop {
    sched.tick();   // 检查所有 Job 的 cron 表达式，若当前时刻匹配则执行闭包
    runtime.block_on(tokio::time::sleep(Duration::from_millis(CONFIG.job_poll_interval_ms())));
}
```

- `sched.tick()` 在每次循环中遍历所有已注册 Job，比对系统时间与 cron 表达式。
- 两次 tick 之间的间隔由 `JOB_POLL_INTERVAL_MS` 控制（默认 30 秒）。
- **设为 0 则全局禁用**：[main.rs:664](src/main.rs#L664) `if CONFIG.job_poll_interval_ms() == 0 { return; }`

### 2.2 Cron 表达式与默认值

所有 cron 表达式都在 [config.rs:539-569](src/config.rs#L539-L569) 的 `jobs` 块中定义，
格式为 6 位秒级 cron（`秒 分 时 日 月 周`）：

| Job | 环境变量 | 默认 cron | 默认频率 |
|-----|---------|-----------|---------|
| Send 过期清理 | `SEND_PURGE_SCHEDULE` | `0 5 * * * *` | 每小时第 5 分钟 |
| 回收站清理 | `TRASH_PURGE_SCHEDULE` | `0 5 0 * * *` | 每天 00:05 |
| 未完成 2FA 通知 | `INCOMPLETE_2FA_SCHEDULE` | `30 * * * * *` | 每分钟第 30 秒 |
| 紧急访问超时授权 | `EMERGENCY_REQUEST_TIMEOUT_SCHEDULE` | `0 7 * * * *` | 每小时第 7 分钟 |
| 紧急访问提醒 | `EMERGENCY_NOTIFICATION_REMINDER_SCHEDULE` | `0 3 * * * *` | 每小时第 3 分钟 |
| 认证请求清理 | `AUTH_REQUEST_PURGE_SCHEDULE` | `30 * * * * *` | 每分钟第 30 秒 |
| Duo 上下文清理 | `DUO_CONTEXT_PURGE_SCHEDULE` | `30 * * * * *` | 每分钟第 30 秒 |
| 事件日志清理 | `EVENT_CLEANUP_SCHEDULE` | `0 10 0 * * *` | 每天 00:10 |
| SSO 认证清理 | `PURGE_INCOMPLETE_SSO_AUTH` | `0 20 0 * * *` | 每天 00:20 |

**禁用方式**：将对应环境变量设为空字符串，注册时跳过（`if !CONFIG.xxx_schedule().is_empty()`）。

### 2.3 任务注册的条件守卫

并非所有 Job 都无条件注册，部分有额外前置条件：

| Job | 额外条件 | 代码位置 |
|-----|---------|---------|
| Duo 上下文清理 | `CONFIG._enable_duo() && !CONFIG.duo_use_iframe()` | [main.rs:725](src/main.rs#L725) |
| 事件日志清理 | `CONFIG.org_events_enabled() && CONFIG.events_days_retain().is_some()` | [main.rs:732-734](src/main.rs#L732-L734) |
| 其余所有 | 仅检查 schedule 非空 | 各 `if !CONFIG.xxx_schedule().is_empty()` |

### 2.4 同一 tick 内的执行顺序

`job_scheduler_ng` 的 [JobScheduler::tick()](https://github.com/BlackDex/job_scheduler/blob/master/src/lib.rs#L327-L330) 实现非常简单：

```rust
pub fn tick(&mut self) {
    for job in &mut self.jobs {
        job.tick();
    }
}
```

这意味着：

1. **遍历顺序严格等于 `sched.add()` 注册顺序**：
   - [main.rs:679-746](src/main.rs#L679-L746) 的注册顺序：send_purge → trash_purge → incomplete_2fa → **emergency_request_timeout** → **emergency_notification_reminder** → auth_request_purge → duo_context_purge → event_cleanup → sso_auth_purge
   - 紧急访问 timeout job 先于 reminder job 被遍历

2. **闭包同步执行，但任务异步提交**：
   - 每个 `job.tick()` 同步执行闭包（在调度线程内）
   - 闭包内 `runtime.spawn(...)` 将异步任务提交到 tokio 后立即返回
   - 因此：**闭包提交顺序** = 注册顺序，但**异步任务实际执行顺序**由 tokio 调度器决定（不确定）

3. **关键边界**：如果两个 Job 的 cron 表达式在**同一秒**命中（例如都设为 `0 5 * * * *`）：
   - 它们的闭包会按注册顺序依次提交到 tokio
   - 但实际并发执行顺序不可预测
   - 对于有依赖关系的 Job（如紧急访问的两个任务），同一时间点命中可能产生竞态

### 2.5 Cron 配置校验

配置校验分为两个阶段：

**阶段一：启动时校验**（[config.rs:1212-1243](src/config.rs#L1212-L1243)）

服务器启动时，在 `validate_config()` 中对所有非空的 cron schedule 做 parse 校验：

```rust
if !cfg.send_purge_schedule.is_empty() && cfg.send_purge_schedule.parse::<Schedule>().is_err() {
    err!("`SEND_PURGE_SCHEDULE` is not a valid cron expression")
}
```

此处 `err!()` 宏的定义见 [error.rs:327-337](src/error.rs#L327-L337)，其行为是 **`return Err(Error::new_msg(msg))`**，即返回 `Result::Err`，而非 panic。

但 `validate_config()` 的返回值 `Result<(), Error>` 最终被消费于 [config.rs:44-47](src/config.rs#L44-L47)：

```rust
rt.block_on(Config::load()).unwrap_or_else(|e| {
    println!("Error loading config:\n  {e:?}\n");
    exit(12)  // 退出码 12
})
```

因此，cron 校验失败的实际效果链是：

```
err!() → return Err(Error) → validate_config() 返回 Err
→ Config::load() 返回 Err → unwrap_or_else → exit(12)
```

**结论**：不是 panic，而是**以退出码 12 正常退出进程**。对运维而言，进程消失但不会产生 panic 回溯栈。

**校验覆盖范围（已校验）**：
- `SEND_PURGE_SCHEDULE`
- `TRASH_PURGE_SCHEDULE`
- `INCOMPLETE_2FA_SCHEDULE`
- `EMERGENCY_NOTIFICATION_REMINDER_SCHEDULE`
- `EMERGENCY_REQUEST_TIMEOUT_SCHEDULE`
- `EVENT_CLEANUP_SCHEDULE`
- `AUTH_REQUEST_PURGE_SCHEDULE`

**校验缺失（未校验，有风险）**：
- `DUO_CONTEXT_PURGE_SCHEDULE`
- `PURGE_INCOMPLETE_SSO_AUTH`

这两个配置项没有启动校验。若配置错误，会在 `schedule_jobs()` 注册时通过 `.parse().unwrap()` 触发**真正的 panic**（因为 `unwrap()` 在 `Err` 上 panic），且 panic 发生在独立 OS 线程 `job-scheduler` 中而非主线程——进程不会因此自动退出，而是调度线程静默崩溃，所有定时任务停止运行。

**阶段二：注册时 parse**（[main.rs:681](src/main.rs#L681) 等各处）

```rust
sched.add(Job::new(CONFIG.send_purge_schedule().parse().unwrap(), || { ... }));
```

理论上启动校验已保证合法性，但仍使用 `unwrap()`。由于 CONFIG 在运行时是只读的，不存在动态修改导致 parse 失败的路径。

### 2.6 紧急访问任务的时间依赖与自定义配置风险

#### 默认配置的时间差设计

默认 cron 配置：
| Job | Cron | 触发时间 |
|-----|------|---------|
| emergency_request_timeout | `0 7 * * * *` | 每小时第 7 分钟 0 秒 |
| emergency_notification_reminder | `0 3 * * * *` | 每小时第 3 分钟 0 秒 |

**注意：reminder 实际先于 timeout 执行（03 < 07），与注释意图相反。**

[main.rs:702-703](src/main.rs#L702-L703) 的注释明确期望 timeout 先执行：

> This job should run before the emergency access reminders job to avoid
> sending reminders for requests that are about to be granted anyway.

但默认 cron 时间安排是 reminder（03）→ timeout（07），即**先提醒，4 分钟后才批准**。

#### 设计意图与实际行为的矛盾

```
理想时序（注释期望）：   timeout ──> reminder（批准的就不提醒）
实际默认时序：         reminder（可能提醒了即将批准的） ──> timeout（批准）
```

这意味着在默认配置下，有些即将被批准的紧急访问请求会被先发送提醒邮件，造成**不必要的邮件打扰**。

#### 自定义时间调整的风险场景

如果用户自定义这两个 cron 表达式，可能出现以下情况：

##### 场景 1：设为同一时间点——reminder 是否仍会提醒？

```bash
EMERGENCY_REQUEST_TIMEOUT_SCHEDULE="0 5 * * * *"
EMERGENCY_NOTIFICATION_REMINDER_SCHEDULE="0 5 * * * *"
```

同一 tick 内，两个闭包按注册顺序依次被 `runtime.spawn()` 提交到 tokio。提交是同步的、非阻塞的，之后两个异步任务在 tokio 调度器中并发运行。关键在于：**两个任务各自独立从连接池获取数据库连接**（`pool.get().await`），各自持有独立的 `DbConn`。

追踪 [emergency_request_timeout_job](src/api/core/emergency_access.rs#L722-L775) 的执行流：

```
1. pool.get().await           → 获取连接 conn_t
2. find_all_recoveries_initiated(&conn_t)  → SELECT ... WHERE status = RecoveryInitiated
3. for each emer:
   a. 判断 recovery_allowed_at <= now
   b. update_access_status_and_save(RecoveryApproved, &now, &conn_t)
      → UPDATE ... SET status=Approved, updated_at=now WHERE uuid=X
   c. 发送邮件
```

追踪 [emergency_notification_reminder_job](src/api/core/emergency_access.rs#L777-L834) 的执行流：

```
1. pool.get().await           → 获取连接 conn_r
2. find_all_recoveries_initiated(&conn_r)  → SELECT ... WHERE status = RecoveryInitiated
3. for each emer:
   a. 判断 final_recovery_reminder_at <= now && next_recovery_reminder_at <= now
   b. update_last_notification_date_and_save(&now, &conn_r)
      → UPDATE ... SET last_notification_at=now, updated_at=now WHERE uuid=X
   c. 发送邮件
```

**两种可能的执行交错**：

**交错 A：timeout 先完成 SELECT，再 UPDATE 了 status**
- reminder 随后做 SELECT，此时该记录 `status = RecoveryApproved`，不再满足 `find_all_recoveries_initiated` 的过滤条件
- **结果：该记录不会出现在 reminder 列表中，不发送提醒邮件** ✓

**交错 B：reminder 先完成 SELECT（此时 status 仍为 RecoveryInitiated）**
- reminder 拿到了包含该记录的列表
- 即使 timeout 随后 UPDATE 了 status，reminder 的列表已经是内存中的快照，不会重新查询
- **结果：reminder 仍会对该记录发送提醒邮件，然后 UPDATE last_notification_at**
- 这封提醒邮件是**不必要的**——因为 timeout 随后就会批准该请求

**结论**：同秒命中时，reminder **可能仍会提醒**（取决于 SELECT 和 UPDATE 的交错时机）。具体来说：
- 如果 timeout 的 UPDATE 在 reminder 的 SELECT 之前完成 → 不提醒
- 如果 reminder 的 SELECT 在 timeout 的 UPDATE 之前完成 → **会提醒**（多余邮件）

两种交错的概率取决于 tokio 调度和数据库响应速度。由于两个任务各自持有独立的 `DbConn`，不存在连接级串行化，交错 B 是完全可能的。

##### 场景 2：timeout 先于 reminder 执行（正确的时序）

```bash
EMERGENCY_REQUEST_TIMEOUT_SCHEDULE="0 2 * * * *"    # 第 2 分钟
EMERGENCY_NOTIFICATION_REMINDER_SCHEDULE="0 5 * * * *"  # 第 5 分钟
```

- 符合注释意图：先批准，再为剩余未批准的发送提醒
- 是推荐的正确配置方式

##### 场景 3：reminder 先于 timeout 执行（默认行为，有浪费）

```bash
EMERGENCY_REQUEST_TIMEOUT_SCHEDULE="0 5 * * * *"    # 第 5 分钟
EMERGENCY_NOTIFICATION_REMINDER_SCHEDULE="0 2 * * * *"  # 第 2 分钟
```

- 先提醒，3 分钟后批准
- 可能发送不必要的提醒（批准前 3 分钟被提醒了）
- 但功能上是正确的，只是用户体验和资源利用上有浪费

##### 场景 4：执行间隔小于单任务执行时间（重叠风险 ⚠️）

如果两个任务的 cron 间隔很近（例如相差 1 分钟），而 reminder 任务执行时间超过 1 分钟：
- timeout 的新实例被 spawn 时，reminder 的旧实例可能仍在运行
- 两者并发操作同一条 EmergencyAccess 记录
- 依赖数据库乐观并发（UPDATE 只改一个字段）+ retry(10) 兜底

#### 字段级隔离的并发保护

为减少写冲突，两个任务采用**只更新各自关心的字段**的设计：

- [update_access_status_and_save](src/db/models/emergency_access.rs#L178-L201)：只 SET `status` + `updated_at`
- [update_last_notification_date_and_save](src/db/models/emergency_access.rs#L203-L219)：只 SET `last_notification_at` + `updated_at`

这是一种乐观的字段级隔离，降低了并发写相互覆盖的概率。但 `updated_at` 字段仍会相互覆盖（后写胜出）。

---

## 3. 过期清理逻辑详解

每个 Job 的清理逻辑分为两层：**API 层入口函数**（从 pool 获取连接）和 **Model 层数据操作**（执行查询与删除）。

### 3.1 Send 过期清理

- **入口**：[purge_sends](src/api/core/sends.rs#L61-L68)
- **Model**：[Send::purge](src/db/models/send.rs#L246-L250)
- **过期判定**：`deletion_date < now()`（[find_by_past_deletion_date](src/db/models/send.rs#L329-L335)）
- **清理方式**：逐条调用 `send.delete()`，会同时删除数据库记录和关联的文件存储
- **触发**：用户访问已过期 Send 时也会拒绝（[post_access](src/api/core/sends.rs#L474)），但不会主动删除

### 3.2 回收站 Cipher 清理

- **入口**：[purge_trashed_ciphers](src/api/core/ciphers.rs#L106-L113)
- **Model**：[Cipher::purge_trash](src/db/models/cipher.rs#L508-L516)
- **过期判定**：`deleted_at < (now - trash_auto_delete_days)`（[find_deleted_before](src/db/models/cipher.rs#L968-L973)）
- **前提配置**：`TRASH_AUTO_DELETE_DAYS` 必须设置，否则 `purge_trash` 直接返回（不清理）
- **注意**：`deleted_at` 是 cipher 被软删除（移入回收站）的时间戳，不是 cipher 创建时间

### 3.3 Auth Request 清理

- **入口**：[purge_auth_requests](src/api/core/accounts.rs#L1702-L1709)
- **Model**：[AuthRequest::purge_expired_auth_requests](src/db/models/auth_request.rs#L181-L188)
- **过期判定**：`creation_date < (now - 15 minutes)`（硬编码 15 分钟，与上游 Bitwarden 对齐）
- **清理方式**：逐条 `auth_request.delete()`

### 3.4 Duo 上下文清理

- **入口**：[purge_duo_contexts](src/api/core/two_factor/duo_oidc.rs)（通过 `main.rs:67` import）
- **Model**：[TwoFactorDuoContext::purge_expired_duo_contexts](src/db/models/two_factor_duo_context.rs#L71-L75)
- **过期判定**：`exp < now_timestamp()`（[find_expired](src/db/models/two_factor_duo_context.rs#L51-L59)）
- **exp 字段**：在 [save](src/db/models/two_factor_duo_context.rs#L28-L49) 时计算为 `now + ttl`

### 3.5 事件日志清理

- **入口**：[event_cleanup_job](src/api/core/events.rs#L325-L337)
- **Model**：[Event::clean_events](src/db/models/event.rs#L336-L348)
- **过期判定**：`event_date < (now - events_days_retain)`
- **特殊之处**：
  - 使用单条 `DELETE ... WHERE event_date < dt` 批量删除，而非逐条
  - 需同时满足 `org_events_enabled` 和 `events_days_retain` 已配置才注册
  - 入口函数也做了二次检查（`events_days_retain().is_none()` 则 return）

### 3.6 SSO 认证清理

- **入口**：直接调用 [SsoAuth::delete_expired](src/db/models/sso_auth.rs#L142-L155)
- **过期判定**：`created_at < (now - SSO_AUTH_EXPIRATION)`，其中 `SSO_AUTH_EXPIRATION` 为 10 分钟（[sso.rs:24-25](src/sso.rs#L24-L25)）
- **特殊之处**：
  - 使用单条 `DELETE ... WHERE created_at < oldest` 批量删除
  - 不经过 API 层封装，直接在 Model 上操作 `DbPool`

### 3.7 未完成 2FA 通知（非清理，是通知）

- **入口**：[send_incomplete_2fa_notifications](src/api/core/two_factor/mod.rs#L243-L284)
- **逻辑**：查找 `login_time < (now - incomplete_2fa_time_limit)` 的记录（[find_logins_before](src/db/models/two_factor_incomplete.rs#L87-L95)），
  发送邮件通知后删除该记录
- **前提**：`incomplete_2fa_time_limit > 0` 且邮件已启用

### 3.8 紧急访问超时授权与提醒（非清理，是状态流转）

- **超时授权**：[emergency_request_timeout_job](src/api/core/emergency_access.rs#L722-L775)
  - 查找所有 `status == RecoveryInitiated` 且 `recovery_initiated_at` 非空的记录
  - 若 `recovery_initiated_at + wait_time_days <= now`，则将状态更新为 `RecoveryApproved`
  - 只更新 `status` 和 `updated_at` 字段（[update_access_status_and_save](src/db/models/emergency_access.rs#L178-L201)）
- **到期提醒**：[emergency_notification_reminder_job](src/api/core/emergency_access.rs#L777-L834)
  - 同样查找 `RecoveryInitiated` 状态的记录
  - 计算最终提醒时间 = `recovery_initiated_at + wait_time_days - 1天`
  - 计算下次提醒时间 = `last_notification_at + 1天`（若无上次通知则为 `now`）
  - 只更新 `last_notification_at` 和 `updated_at` 字段（[update_last_notification_date_and_save](src/db/models/emergency_access.rs#L203-L219)）

---

## 4. 重复执行保护

### 4.1 调度器层面：cron 粒度天然防重

`job_scheduler_ng` 的 cron 匹配是**时间点匹配**而非"至少间隔"语义。
以 `0 5 * * * *`（每小时第 5 分钟）为例，只有在 `:05:00` 这一个 tick 才会触发。
由于 `JOB_POLL_INTERVAL_MS` 默认 30 秒，tick 频率远高于 cron 最小粒度（秒级），
因此每个 cron 时间点只会被命中一次。

**但存在风险场景**：如果某次 tick 执行时间超过一个 poll interval（如数据库慢查询），
`sched.tick()` 不会为"跳过"的时间点补触发——这是 `job_scheduler_ng` 的设计，它只匹配"当前 tick 时刻"。

### 4.2 运行时层面：`runtime.spawn` 即忘即弃

所有 Job 闭包的核心模式：

```rust
sched.add(Job::new(cron, || {
    runtime.spawn(async_fn(pool.clone()));  // 非阻塞，提交到 tokio 后立即返回
}));
```

`runtime.spawn` 将异步任务提交到 tokio 调度器后**立即返回**，不会等待任务完成。
这意味着：

- **同一种 Job 的多次触发可能并行执行**：如果 `purge_sends` 执行时间超过 1 小时（默认 cron 间隔），
  下一个 cron 时间点到来时，新的 `purge_sends` 会被 spawn，与前一次并行。
- **没有互斥锁或原子标记**：代码中不存在 `Mutex`、`AtomicBool` 或数据库级的 advisory lock 来阻止重叠执行。

### 4.3 业务层面的幂等性

虽然没有显式的防重入机制，但大部分 Job 在业务上是**幂等的**：

| Job | 幂等性来源 |
|-----|----------|
| Send 清理 | 删除后下次查询自然查不到，重复删除仅产生 `Ok(())` |
| Cipher 回收站清理 | 同上，`delete` 对已删除记录是空操作 |
| Auth Request 清理 | 同上 |
| Duo 上下文清理 | 同上 |
| Event 清理 | 批量 `DELETE WHERE date < X`，幂等 |
| SSO Auth 清理 | 批量 `DELETE WHERE created_at < X`，幂等 |
| 2FA 通知 | 邮件发送 + 删除记录：若记录已被删除则查不到；若邮件已发送但未删除，**可能重复发送邮件** |
| 紧急访问超时 | 状态从 `RecoveryInitiated` → `RecoveryApproved`，已批准的不会被 `find_all_recoveries_initiated` 查到 |
| 紧急访问提醒 | 依赖 `last_notification_at` 判断，但**并发时可能发送重复提醒邮件** |

### 4.4 紧急访问两个 Job 之间的协作

[main.rs:702-703](src/main.rs#L702-L703) 注释明确说明：

> This job should run before the emergency access reminders job to avoid
> sending reminders for requests that are about to be granted anyway.

两个 Job 的 cron 时间差为 4 分钟（timeout 在 `:07`，reminder 在 `:03`），
**但 timeout 实际在 reminder 之后执行**（07 > 03），所以这里的注释描述的是"应该先运行"的意图，
而非当前的 cron 时间安排。两者使用 `find_all_recoveries_initiated` 查询同一组数据，
通过**只更新各自关心的字段**（[update_access_status_and_save](src/db/models/emergency_access.rs#L178-L201) 只改 `status`+`updated_at`，
[update_last_notification_date_and_save](src/db/models/emergency_access.rs#L203-L219) 只改 `last_notification_at`+`updated_at`）来减少写冲突。

---

## 5. 并发清理边界

### 5.1 单实例内的并发

**调度线程**与 **Rocket 异步运行时**是两个独立的 tokio Runtime：

1. **调度线程**：在 `thread::Builder::new().name("job-scheduler")` 创建的 OS 线程中，
   有自己的 `tokio::runtime::Runtime`（[main.rs:669](src/main.rs#L669)）
2. **Rocket 运行时**：由 Rocket 框架管理的 tokio 异步运行时，处理 HTTP 请求

两者通过 `DbPool`（连接池）共享数据库访问。连接池本身保证了并发连接数的上限，
但不提供应用级的事务隔离。

### 5.2 数据库操作的事务范围

所有清理操作**没有使用显式事务**（`BEGIN / COMMIT`）：

- **逐条删除型**（Send、Cipher、AuthRequest、DuoContext）：`SELECT → for each: DELETE`
  - SELECT 和 DELETE 之间没有锁保护，其他连接可能在间隙中修改数据
  - 对于 SQLite，`db_run!` 宏内的单条 SQL 是原子的（SQLite 内核保证）
  - 对于 PostgreSQL/MySQL，单条 `DELETE WHERE uuid = X` 是原子的，但 `SELECT + DELETE` 组合不是

- **批量删除型**（Event、SsoAuth）：单条 `DELETE WHERE date < X`
  - 这是一条原子 SQL，在数据库层面是安全的

- **状态更新型**（EmergencyAccess）：`UPDATE ... SET status = X WHERE uuid = Y`
  - 使用了 `crate::util::retry` 重试机制（[emergency_access.rs:190-197](src/db/models/emergency_access.rs#L190-L197)），
    最多重试 10 次，说明开发者已考虑到并发冲突的可能性

### 5.3 多实例部署的竞态风险

Vaultwarden 的 Job 调度器是**进程本地**的（`JobScheduler::new()` 创建纯内存实例），
没有任何分布式锁或数据库级的排他机制。如果部署多个 Vaultwarden 实例共享同一数据库：

| 风险 | 说明 |
|------|------|
| **重复执行** | 每个实例都会独立运行所有 Job，同一时刻 N 个实例会执行 N 次相同的清理 |
| **邮件重复** | 2FA 通知、紧急访问提醒可能被多个实例同时触发，导致用户收到重复邮件 |
| **数据竞争** | 两个实例同时对同一条 EmergencyAccess 记录做 `update_access_status_and_save`，后写覆盖前写 |
| **删除冲突** | 对已删除记录重复 DELETE 通常无害（返回 affected_rows = 0），但不会报错 |

**官方推荐**：多实例部署时，只在一个实例上启用 Job 调度器（通过 `JOB_POLL_INTERVAL_MS=0` 禁用其余实例的调度器）。

### 5.4 连接池耗尽边界

所有 Job 入口函数的统一模式：

```rust
if let Ok(conn) = pool.get().await {
    // 正常处理
} else {
    error!("Failed to get DB connection while ...");
}
```

当连接池耗尽时，`pool.get()` 返回 `Err`，Job 会**静默跳过**本次执行（仅打 error 日志），
不会排队等待或重试。这意味着在数据库压力较大时，维护任务可能被连续跳过。

---

## 6. 总结：各 Job 的清理边界速查表

| Job | 过期条件 | 清理粒度 | 事务保护 | 并发安全 | 幂等 |
|-----|---------|---------|---------|---------|------|
| Send 清理 | `deletion_date < now` | 逐条 DELETE | 无 | ✅ 幂等 | ✅ |
| Cipher 回收站 | `deleted_at < now - N days` | 逐条 DELETE | 无 | ✅ 幂等 | ✅ |
| Auth Request | `creation_date < now - 15min` | 逐条 DELETE | 无 | ✅ 幂等 | ✅ |
| Duo Context | `exp < now_timestamp` | 逐条 DELETE | 无 | ✅ 幂等 | ✅ |
| Event 清理 | `event_date < now - N days` | 批量 DELETE | 单条 SQL 原子 | ✅ | ✅ |
| SSO Auth | `created_at < now - 10min` | 批量 DELETE | 单条 SQL 原子 | ✅ | ✅ |
| 2FA 通知 | `login_time < now - N min` | 逐条：发邮件+DELETE | 无 | ⚠️ 可能重复发邮件 | ⚠️ |
| 紧急访问超时 | `initiated_at + wait_days <= now` | 逐条 UPDATE status | retry(10) | ⚠️ 多实例写冲突 | ✅（状态单向流转） |
| 紧急访问提醒 | `final_reminder <= now && next_reminder <= now` | 逐条 UPDATE notification_at | retry(10) | ⚠️ 可能重复发邮件 | ⚠️ |
