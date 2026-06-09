# Vaultwarden 图标缓存语义核对（代码级 Followup Fix）

本文是对 `vaultwarden-icon-cache-followup.md` 的修正与补充，完全以代码实际执行路径为准，聚焦：**过期 `.miss` 删除失败后为什么会反复触发**、以及缓存刷新语义中是否存在同类"检测 → 操作"冲突。

---

## 一、过期 `.miss` 删除失败后的循环机制（代码级推演）

### 1.1 触发条件

满足以下两个条件即进入循环：

1. 磁盘上存在 `{domain}.png.miss` 文件，且 `age ≥ icon_cache_negttl`（即已过期）
2. `operator.delete()` 持续失败（例如存储后端权限只读、S3 bucket 设为只读、OpenDAL operator 配置错误但恰好 `stat` 可读等）

### 1.2 单请求内的完整调用链

[icon_is_negcached](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L206-L228)：

```rust
async fn icon_is_negcached(path: &str) -> bool {
    let miss_indicator = path.to_owned() + ".miss";
    // 步骤 A：过期检测
    let expired = file_is_expired(&miss_indicator, CONFIG.icon_cache_negttl()).await;

    match expired {
        Ok(true) => {
            // 步骤 B：惰性清理（关键：结果不影响返回值）
            match CONFIG.opendal_operator_for_path_type(&PathType::IconCache) {
                Ok(operator) => {
                    if let Err(e) = operator.delete(&miss_indicator).await {
                        error!("Could not remove negative cache indicator for icon {path:?}: {e:?}");
                        // 删除失败：仅记日志，状态不变，继续返回 false
                    }
                }
                Err(e) => error!("Could not remove negative cache indicator for icon {path:?}: {e:?}"),
            }
            false  // 无论删除成功与否，一律认为"负缓存不再命中"
        }
        Ok(false) => true,
        Err(_)    => false,
    }
}
```

配合 [file_is_expired](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L196-L204)：

```rust
async fn file_is_expired(path: &str, ttl: u64) -> Result<bool, Error> {
    let operator = CONFIG.opendal_operator_for_path_type(&PathType::IconCache)?;
    let meta = operator.stat(path).await?;
    let modified = meta.last_modified()...?;
    let age = SystemTime::now().duration_since(modified.into())?;
    Ok(ttl > 0 && ttl <= age.as_secs())
}
```

### 1.3 为什么会在后续请求中反复触发

核心矛盾是：**步骤 A 的过期判定基于文件 mtime（纯读），步骤 B 的清理是独立写操作，写失败不回写状态，也不改变返回值**。

逐轮推演：

| 请求轮次 | 步骤 A：`file_is_expired` | 步骤 B：`operator.delete` | 返回值 | 请求后续行为 | 磁盘状态 |
|---|---|---|---|---|---|
| 第 1 次 | `.miss` 存在且 `age ≥ negttl` → `Ok(true)` | 失败（打 error 日志） | `false`（放行） | 走正缓存→下载；若下载再次失败则尝试写 `.miss`（也可能失败） | `.miss` 仍然存在，mtime 未变 |
| 第 2 次 | `.miss` 仍存在，mtime 未变 → `age ≥ negttl` → `Ok(true)` | 再次失败（再次打 error 日志） | `false`（再次放行） | 同上 | `.miss` 仍然存在 |
| 第 N 次 | 同上 | 同上（每次一条 error 日志） | 同上 | 同上 | 同上 |

只要删除操作持续失败，这个循环不会自止。`save_icon` 写新 `.miss` 也失败的情况下更不会终止——连文件覆盖更新 mtime 的机会都没有。

### 1.4 循环的实际后果

- **日志刷盘**：每次该域名请求产生一条 `error!` 级日志。若该域名被高频访问（例如浏览器自动预取），可能产生大量错误日志，遮蔽真正问题。
- **放行语义与事实不一致**：函数返回 `false` 表示"没有命中负缓存"，但磁盘上 `.miss` 仍在。对后续请求的"放行"是自欺欺人——下一轮同样会检测到过期并再次尝试删除。
- **注意不会无限阻塞**：每次请求只尝试一次 delete，不会在请求内重试。所以是"每次请求一条 error log + 放行"的 N 轮循环，不是单次请求内的死循环。

---

## 二、同类"检测 → 操作"冲突全景扫描

按相同模式（过期/存在检测 → 独立写操作 → 写失败不回滚状态/不影响语义）扫描整条链路，发现 3 处额外冲突。

### 2.1 冲突二：正缓存过期 + `save_icon` 写失败 → 每次请求重新下载

对应代码：[get_cached_icon](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L180-L194) + [save_icon](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L572-L584)。

触发条件：
1. 磁盘上存在旧的 `{domain}.png`，`age ≥ icon_cache_ttl`（已过期）
2. 下载能成功，但 `operator.write()` 持续失败（例如存储后端只读）

推演：

| 请求轮次 | `icon_is_expired` | `get_cached_icon` | `download_icon` | `save_icon(write)` | 客户端收到 | 磁盘状态 |
|---|---|---|---|---|---|---|
| 第 1 次 | `true`（过期） | `None`（不读） | 成功（真实下载） | 失败（打 warn 日志） | 新图标（内存中） | 仍是旧 `.png`，mtime 未变 |
| 第 2 次 | 仍 `true`（mtime 没变） | `None` | 再次真实下载 | 再次失败（再次 warn） | 新图标 | 仍是旧 `.png` |
| 第 N 次 | 同上 | 同上 | 同上 | 同上 | 同上 | 同上 |

后果：
- **每次请求都发起真实网络下载**，绕过了"下载→缓存→复用"的核心设计。若该图标被高频访问，外部网络成本和目标站点压力都显著增加。
- 与 `.miss` 删除循环类似，写失败是静默降级的——只打 warn 日志，不向上层传递。管理员只能通过日志或异常的出站流量发现。
- 与 `.miss` 循环的不同点：客户端每次都能收到正确的最新图标（因为内存里有下载结果），问题只出在"缓存没有被刷新"这一侧。

### 2.2 冲突三：系统时间回拨导致正负缓存不对称失效

对应代码：`file_is_expired` 中的

```rust
let age = SystemTime::now().duration_since(modified.into())?;
```

`duration_since` 在 `modified > now`（系统时间回拨）时返回 `Err`。这个 `Err` 在两条路径上的处理方式完全相反：

| 路径 | 代码 | `Err` 处理 | 效果 |
|---|---|---|---|
| 负缓存 | [icons.rs:226](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L226-L226) `Err(_) => false` | 视为"未命中负缓存" | 放行，允许继续查正缓存或下载 |
| 正缓存 | [icons.rs:232](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L232-L232) `expired.unwrap_or(true)` | 视为"已过期" | 不返回缓存，触发重新下载 |

系统时间回拨后的实际行为：
- `.miss` 文件（负缓存）被完全忽略，相当于"负缓存永不存在"
- `.png` 文件（正缓存）被强制视为过期，即使是刚写入的

两者叠加的效果：**时间回拨后，所有图标请求都会跳过缓存层，直连外部下载**。这本身不会形成循环，但属于缓存语义的不对称崩溃——同类原因（stat/time 失败）在正负压路径上给出了相反判定。

### 2.3 冲突四：TTL 过大导致 Cached Responder panic

对应代码：[util.rs:254](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/util.rs#L254-L254)

```rust
let expiry_time = time_now + chrono::TimeDelta::try_seconds(self.ttl.try_into().unwrap()).unwrap();
```

两层 `unwrap()` 的潜在问题：
1. `self.ttl` 是 `u64`，`try_into()` 目标是 `i64`（`chrono::TimeDelta::try_seconds` 的签名参数）。
2. 当 `u64` 值 > `i64::MAX`（即 > 9,223,372,036,854,775,807 秒，约 292 亿年）时，第一次 `try_into().unwrap()` panic。
3. 即使成功转成 `i64`，`TimeDelta::try_seconds` 也会在超范围时返回 `None`，触发第二次 `.unwrap()` panic。

配置项 `icon_cache_ttl` 和 `icon_cache_negttl` 在 [config.rs](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/config.rs) 中声明为 `u64`，加载时只校验了基本类型，没有上限校验。

实际触发概率极低（需要手工把 TTL 配到 292 亿年以上），但作为通用 `Cached<R>` Responder，它也可能被其他调用方传入非常大的 TTL 值。

### 2.4 冲突五（确认不是循环但需注明）：TTL=0 两层语义相反

此条在前序文档已提及，这里用代码再次精确对齐：

| 层级 | 代码 | TTL=0 的实际含义 |
|---|---|---|
| 服务端磁盘缓存 | `Ok(ttl > 0 && ttl <= age.as_secs())` 中 `ttl > 0` 为 false | **永不过期**（正缓存永远命中；负缓存永远拦截） |
| HTTP 响应缓存 | `format!("public, immutable, max-age={}", self.ttl)` | **立即过期**（`max-age=0`，浏览器每次重验证） |

不是循环冲突，但属于**同一个配置值在两层缓存中语义相反**，容易造成运维误判——例如把 TTL 设为 0 想让浏览器不缓存，结果服务端磁盘缓存反而永不刷新。

---

## 三、所有冲突总表

| # | 场景 | 检测操作 | 写操作 | 写失败/异常后的行为 | 后果 | 是否循环 |
|---|---|---|---|---|---|---|
| 1 | `.miss` 过期 | `operator.stat` + mtime 比较 | `operator.delete` | 打 error 日志，返回 `false`（放行） | 每次请求一条 error log，持续尝试删除 | 是（N 轮，每轮 1 次） |
| 2 | `.png` 过期后重新下载 | 同上 | `operator.write` | 打 warn 日志，返回内存数据给客户端 | 每次请求都真实下载，缓存形同虚设 | 是（N 轮，每轮 1 次下载） |
| 3 | 系统时间回拨 | `SystemTime::duration_since` 返回 Err | 无 | 负缓存：放行（`Err→false`）；正缓存：强制过期（`unwrap_or(true)`） | 同类异常在正负压路径判定相反，所有缓存失效 | 否（一次性崩溃） |
| 4 | TTL > `i64::MAX` | 无 | 无（纯计算） | `ttl.try_into().unwrap()` panic | 请求线程 panic，可能产生 500 | 否（单次 panic） |
| 5 | TTL = 0 | 无 | 无 | 服务端磁盘=永不过期；HTTP 响应=立即过期 | 同一配置语义相反，运维易误判 | 否（语义级冲突） |

---

## 四、代码级修复方向提示（非本次任务执行，仅供参考）

仅列出能直接对应到代码行的修复思路，不做实现：

- **冲突 1**：删除失败时把 `.miss` 当作仍命中（返回 `true`），或尝试用"touch 更新 mtime"代替删除，避免每轮重复 stat+delete。
- **冲突 2**：`save_icon` 失败时向上传播至少一个标记位，让调用方在内存中保留"这次下载过"的短 TTL 记忆，避免下一轮立即再次下载。
- **冲突 3**：正负缓存路径统一 `Err` 处理语义（建议都按"保守放行"或"保守失效"，但要一致）。
- **冲突 4**：对 `u64→i64` 做 `unwrap_or(i64::MAX)` 或饱和转换，避免 panic。
- **冲突 5**：配置加载时对 TTL=0 打印警告日志，或在文档中显式强调双层语义。
