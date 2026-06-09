# Vaultwarden 图标缓存语义核对（代码级 Followup）

本文从 Rust 源码出发，逐行核对三个具体问题的语义。所有结论均可回溯到代码。

---

## 一、缓存时间（TTL）为零的含义

Vaultwarden 存在**两个独立的缓存层**，各自对 TTL=0 的解释不同。

### 1.1 服务端磁盘缓存层：TTL=0 = 永不过期

核心判定函数 [icons.rs:196-204](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L196-L204)：

```rust
async fn file_is_expired(path: &str, ttl: u64) -> Result<bool, Error> {
    // ... 取文件 mtime，计算 age = now - mtime ...
    Ok(ttl > 0 && ttl <= age.as_secs())
}
```

表达式 `ttl > 0 && ttl <= age.as_secs()` 的真值表：

| `ttl` | `age` 关系 | 返回值 | 语义 |
|---|---|---|---|
| 0 | 任意 | `false` | **文件永不过期** |
| > 0 但 ≤ age | `ttl <= age` 为 true | `true` | 文件已过期 |
| > 0 且 > age | `ttl <= age` 为 false | `false` | 文件未过期 |

#### 正缓存（`icon_cache_ttl=0`）的效果

通过调用链：`icon_is_expired` → `file_is_expired(..., CONFIG.icon_cache_ttl())`

[icons.rs:230-233](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L230-L233)：
```rust
async fn icon_is_expired(path: &str) -> bool {
    let expired = file_is_expired(path, CONFIG.icon_cache_ttl()).await;
    expired.unwrap_or(true)  // 注：stat 失败时默认算"已过期"
}
```

当 `icon_cache_ttl=0`：`file_is_expired` 返回 `Ok(false)` → `icon_is_expired` 返回 `false` → [icons.rs:182](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L182-L184) 的早期返回不触发 → 继续尝试读取磁盘文件。

**结果**：磁盘上的 `{domain}.png` 永远不会因 TTL 过期而被淘汰，服务端每次都会从磁盘返回。这正是 `disable_icon_download` 文档注释所依赖的语义 [config.rs:612-615](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/config.rs#L612-L615)：

> `$ICON_CACHE_TTL must also be set to 0; otherwise, the existing icons will be deleted eventually, but won't be downloaded again.`

（注：代码里并没有主动"删除"过期文件，而是过期后下一次请求不会命中缓存，从而触发重新下载——当下载被禁用时就永远返回 fallback。所以注释用了"deleted eventually"这个说法。）

#### 负缓存（`icon_cache_negttl=0`）的效果

通过调用链：`icon_is_negcached` → `file_is_expired(..., CONFIG.icon_cache_negttl())`

[icons.rs:206-228](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L206-L228)：
```rust
match expired {
    Ok(true)  => { 删除 .miss; false }  // 过期 → 删除标记，允许重试
    Ok(false) => true,                   // 未过期 → 负缓存命中
    Err(_)    => false,                  // 文件不存在 → 允许重试
}
```

当 `icon_cache_negttl=0`：`file_is_expired` 返回 `Ok(false)` → `icon_is_negcached` 返回 `true` → [icons.rs:145-147](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L145-L147) 直接返回 `None`，连正缓存都不检查。

**结果**：只要 `.miss` 文件存在，就**永久命中负缓存**，永远不会重试下载，也永远不会返回可能同时存在的正缓存。这是一个反直觉的边缘行为——`negttl=0` 的实际效果是"负缓存永不过期"而不是"不使用负缓存"。

### 1.2 HTTP 响应缓存层：TTL=0 = 不缓存

[util.rs:242-257](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/util.rs#L242-L257)：

```rust
let cache_control_header = if self.is_immutable {
    format!("public, immutable, max-age={}", self.ttl)
} else {
    format!("public, max-age={}", self.ttl)
};
res.set_raw_header("Cache-Control", cache_control_header);

let expiry_time = time_now + chrono::TimeDelta::try_seconds(self.ttl.try_into().unwrap()).unwrap();
res.set_raw_header("Expires", format_datetime_http(&expiry_time));
```

当调用方传入 `ttl=0`：
- `Cache-Control: public, immutable, max-age=0`（或没有 immutable）
- `Expires: <当前时间>`

标准 HTTP 语义：`max-age=0` 告诉客户端响应立即过期，每次都应重新验证。`immutable` 与 `max-age=0` 组合没有实际意义（RFC 8246 规定 immutable 只在 `max-age > 0` 时有用），但浏览器通常会忽略 immutable 并按 `max-age=0` 处理。

### 1.3 小结：两层 TTL=0 的语义对照

| 缓存层 | TTL=0 语义 | 实现位置 |
|---|---|---|
| 服务端磁盘正缓存 | 永不过期（始终命中） | `file_is_expired` 中的 `ttl > 0` 判断 |
| 服务端磁盘负缓存 | 永不失效（永远拦截） | 同上 |
| HTTP 客户端缓存 | 立即过期（每次都重新请求） | `Cached<R>` Responder |

---

## 二、正负缓存并存时的命中优先级

### 2.1 代码顺序 = 优先级

[icons.rs:141-152](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L141-L152)：

```rust
async fn get_icon(domain: &str) -> Option<(Vec<u8>, String)> {
    let path = format!("{domain}.png");

    // 第 1 步：检查负缓存
    if icon_is_negcached(&path).await {
        return None;
    }

    // 第 2 步：检查正缓存
    if let Some(icon) = get_cached_icon(&path).await {
        // ...
        return Some((icon, icon_type.to_owned()));
    }
    // ...
}
```

**优先级结论**：负缓存（`.miss` 标记）优先于正缓存（`.png` 图标文件）。

### 2.2 一个反直觉的场景

如果磁盘上同时存在：
- `example.com.png`（有效正缓存）
- `example.com.png.miss`（未过期负缓存）

执行顺序：
1. `icon_is_negcached("example.com.png")` → 检测到 `.miss` 且未过期 → 返回 `true`
2. `get_icon` 立即 `return None`
3. `get_cached_icon` **根本不会被调用**
4. 最终外层 [icon_internal:137](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L137-L137) 返回 fallback 图标

**即使正缓存完好且未过期，负缓存也会将其完全屏蔽。**

### 2.3 为什么会出现并存？

代码中不保证原子地删除旧正缓存并写入负缓存。出现并存的可能路径：

1. 首次请求：下载成功，写入 `example.com.png`（正缓存）
2. 一段时间后：正缓存自然过期（超过 `icon_cache_ttl`）
3. 再次请求：`icon_is_negcached` → `.miss` 不存在 → false；`get_cached_icon` → `icon_is_expired` → true → None；执行下载
4. 下载失败：写入 `example.com.png.miss`（负缓存）
5. 此时磁盘上同时存在过期的正缓存 `.png` 和有效的负缓存 `.miss`

正缓存文件不会被自动删除（代码里没有清理逻辑，只判断过期），所以并存是常态而非异常。

---

## 三、失败标记（`.miss`）过期后的重试流程

### 3.1 完整时序

以 T0 时刻下载失败创建 `.miss`，T1 时刻（已过 `icon_cache_negttl`）再次请求为例：

#### 阶段 1：创建负缓存

```
T0: 请求 example.com 图标
    → get_icon("example.com")
    → icon_is_negcached("example.com.png") → .miss 不存在 → Err(_) → false (允许)
    → get_cached_icon("example.com.png")   → .png 不存在 → None
    → download_icon("example.com")         → 网络失败
    → save_icon("example.com.png.miss", vec![])  ← 写入空文件
    → return None
    → 外层返回 fallback.png (HTTP Cached, negttl)
```

[icons.rs:172-175](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L172-L175)。

#### 阶段 2：`negttl` 期间重复请求（命中负缓存）

```
T0 < t < T0+negttl: 任意请求
    → icon_is_negcached("example.com.png")
        → file_is_expired("example.com.png.miss", negttl)
            → age < negttl → ttl>0 && ttl<=age → false
            → 返回 Ok(false)
        → match Ok(false) => true
    → return None （不读正缓存，不下载）
    → 返回 fallback.png (HTTP Cached, negttl)
```

#### 阶段 3：`.miss` 过期后首次请求（惰性清理 + 重试）

```
T1 ≥ T0+negttl: 首次请求
    → icon_is_negcached("example.com.png")
        → file_is_expired("example.com.png.miss", negttl)
            → age ≥ negttl → ttl>0 && ttl<=age → true
            → 返回 Ok(true)
        → match Ok(true):
            → operator.delete("example.com.png.miss").await  ← 惰性删除
            → 返回 false （负缓存不再命中）
    → 继续执行 get_cached_icon("example.com.png")
        → 如果 .png 存在且未过期 → 返回缓存图标
        → 如果 .png 不存在或已过期 → None
    → 若正缓存也未命中：
        → download_icon("example.com")  ← 真正的重试下载
            → 成功 → save_icon("example.com.png", data) → 返回图标
            → 失败 → save_icon("example.com.png.miss", vec![]) → 返回 None
```

核心代码 [icons.rs:210-221](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L210-L221)：

```rust
// No longer negatively cached, drop the marker
Ok(true) => {
    match CONFIG.opendal_operator_for_path_type(&PathType::IconCache) {
        Ok(operator) => {
            if let Err(e) = operator.delete(&miss_indicator).await {
                error!("Could not remove negative cache indicator for icon {path:?}: {e:?}");
            }
        }
        Err(e) => error!("Could not remove negative cache indicator for icon {path:?}: {e:?}"),
    }
    false
}
```

### 3.2 关键语义点

| 语义点 | 代码位置 | 说明 |
|---|---|---|
| **惰性过期** | `icon_is_negcached` 在每次请求时检查，无后台任务 | `.miss` 过期不会自动消失，要等到下一次相关请求才触发清理 |
| **删除失败容错** | 删除 `.miss` 失败时只打 error log，不改变返回值 `false` | 即使删不掉，也认为负缓存已失效，本次请求仍会继续尝试正缓存/下载。**但下次请求会再次看到未过期的 `.miss`，再次尝试删除**——可能陷入反复打错误日志的循环 |
| **删除与后续操作非原子** | `.miss` 的 delete、`.png` 的 read、`.png/.miss` 的 write 是三次独立 OpenDAL 调用 | 并发场景下可能出现：请求 A 刚删除 `.miss`，请求 B 也走到负缓存检查（见不到 `.miss`）→ 两个请求同时执行下载。最终谁先写入谁覆盖（OpenDAL write 默认覆盖） |
| **重试后又失败会重建 `.miss`** | [icons.rs:173-174](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L173-L174) | 过期后的首次重试如果再次失败，会重新写入 `.miss`，重新开始一个 `negttl` 周期的静默期 |

### 3.3 重试流程的状态机

```
         ┌──────────────────────────────────────────────────────┐
         │                                                      │
         ▼                                                      │
   ┌──────────┐   .miss不存在或已过期    ┌───────────────┐       │
   │ (无状态) │ ──────────────────────► │ 尝试正缓存    │       │
   └──────────┘                         └───────┬───────┘       │
         ▲                                      │               │
         │                              命中    │  未命中        │
         │                                      ▼               │
         │                              ┌───────────────┐       │
         │                              │ 返回缓存图标  │       │
         │                              └───────────────┘       │
         │                                      │               │
         │                                      ▼ 下载失败(非黑名单) │
         │                              ┌───────────────┐       │
         │                              │ download_icon │       │
         │                              └───────┬───────┘       │
         │                                      │               │
         │                              ┌───────┴───────┐       │
         │                              ▼               ▼       │
         │                        ┌──────────┐   ┌──────────┐  │
         │                        │  成功    │   │  失败    │  │
         │                        └────┬─────┘   └────┬─────┘  │
         │                             │              │        │
         │                     写{domain}.png    写{domain}.png.miss
         │                             │              │        │
         │                             ▼              ▼        │
         │                        ┌──────────┐   ┌──────────┐  │
         └────────────────────────┤ 返回图标 │   │ 负缓存期 │◄─┘
                                  └──────────┘   └──────────┘
                                                    negttl 内
                                                    请求直接
                                                    返回fallback
```

---

## 四、核对结论汇总

| 问题 | 代码级结论 |
|---|---|
| **TTL=0 在服务端磁盘缓存** | `ttl > 0 && ttl <= age` 短路为 false → 永不过期。正缓存：始终命中；负缓存：永远拦截，永不重试 |
| **TTL=0 在 HTTP 响应层** | `max-age=0` + `Expires=当前时间` → 要求浏览器每次重验证。与服务端语义相反 |
| **正负缓存优先级** | 负缓存先检查（代码第 145 行 vs 第 149 行），命中后直接 return None，正缓存完全被短路 |
| **并存时行为** | `.miss` 有效 → 返回 fallback，即使 `.png` 存在且新鲜 |
| **`.miss` 过期后流程** | 惰性检测到过期 → 删除 `.miss` → 走正常链路（查正缓存 → 必要时下载）→ 若下载再次失败则重建 `.miss` 重启静默期 |
| **删除 `.miss` 失败** | 仅记日志，本次仍放行（返回 false）。下一次请求再次遇到过期 `.miss`，再次尝试删除——可能形成日志刷屏循环 |
