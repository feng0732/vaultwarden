# Vaultwarden 图标缓存与 Favicon 抓取协作分析

## 一、整体架构概览

Vaultwarden 的图标系统支持两种模式：**内部抓取**（`internal`）和**外部服务重定向**（`bitwarden`/`duckduckgo`/`google`/自定义 URL），由配置项 `ICON_SERVICE` 决定。

路由挂载入口在 [main.rs:590](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/main.rs#L590-L590)：
```rust
.mount([basepath, "/icons"].concat(), api::icons_routes())
```

根据配置选择路由，见 [icons.rs:27-33](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L27-L33)：
```rust
pub fn routes() -> Vec<Route> {
    if CONFIG.icon_service().as_str() == "internal" {
        routes![icon_internal]
    } else {
        routes![icon_external]
    }
}
```

---

## 二、相关配置项

在 [config.rs](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/config.rs) 中定义：

| 配置项 | 默认值 | 含义 |
|---|---|---|
| `icon_service` | `"internal"` | 图标服务模式 |
| `icon_cache_folder` | `$DATA_FOLDER/icon_cache` | 磁盘缓存目录 |
| `icon_cache_ttl` | `2_592_000`（30天） | 成功缓存的有效期（秒） |
| `icon_cache_negttl` | `259_200`（3天） | 失败缓存的有效期（秒） |
| `icon_download_timeout` | `10` | 单图标下载超时（秒） |
| `icon_redirect_code` | `302` | 外部服务重定向状态码 |
| `disable_icon_download` | `false` | 完全禁用外部下载，只从缓存返回 |

---

## 三、代码路径分析

### 3.1 请求入口

两种模式共享相同的 URL 形式：`GET /icons/<host>/icon.png`

#### 模式 A：外部服务重定向 `icon_external`

[icons.rs:85-109](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L85-L109)

```
请求 → 校验 host 合法性 → 检查黑名单/非全局IP
     ├─ 非法/被拦截 → 返回空 Redirect（Cached, negttl）
     └─ 合法 → 构造外部 URL（用 {} 替换域名）→ 根据配置返回 301/302/307/308 重定向（Cached, ttl）
```

关键点：
- 路由名 `icon_external` 被 [util.rs:39-76](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/util.rs#L39-L76) 的 `AppHeaders::on_response` Fairing 特殊识别，跳过 `Cross-Origin-Resource-Policy: same-origin` 响应头，避免 Bitwarden Desktop 客户端下载失败。

#### 模式 B：内部抓取 `icon_internal`

[icons.rs:111-139](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L111-L139)

```
请求 → 校验 host 合法性 → 检查黑名单/非全局IP
     ├─ 非法/被拦截 → 返回内置 fallback-icon.png（Cached, negttl）
     └─ 合法 → 调用 get_icon(domain)
            ├─ 成功 → 返回图标数据 + 正确 Content-Type（Cached, ttl）
            └─ 失败 → 返回内置 fallback-icon.png（Cached, negttl）
```

内置 fallback 图标是编译时嵌入的 `fallback-icon.png`，见 [icons.rs:113](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L113-L113)。

---

### 3.2 核心图标获取 `get_icon`

[icons.rs:141-178](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L141-L178)

这是缓存层与下载层协作的核心函数，执行顺序如下：

```
1. 检查负缓存（.miss 文件）→ 命中则直接返回 None（降级）
2. 检查正缓存（.png 文件）→ 命中且未过期则返回缓存数据
3. 如果 disable_icon_download=true → 返回 None（降级）
4. 执行 download_icon(domain) 下载
     ├─ 成功 → 保存到磁盘缓存（{domain}.png）→ 返回图标
     └─ 失败
          ├─ CustomHttpClientError（黑名单/非全局IP）→ 不写负缓存，直接返回 None
          └─ 其他错误 → 写负缓存标记（{domain}.png.miss 空文件）→ 返回 None
```

---

### 3.3 正缓存（成功缓存）机制

#### 读缓存：`get_cached_icon`

[icons.rs:180-194](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L180-L194)

```
1. 调用 icon_is_expired(path) 检查文件是否过期
   └─ 过期 → 返回 None（触发重新下载）
2. 通过 OpenDAL Operator 读取 {icon_cache_folder}/{domain}.png
   ├─ 成功 → 返回 Vec<u8>
   └─ 失败 → 返回 None（触发重新下载）
```

#### 过期判定：`icon_is_expired` / `file_is_expired`

[icons.rs:230-233](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L230-L233) 和 [icons.rs:196-204](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L196-L204)

逻辑：
- 读取文件的 `last_modified` 时间
- `age = now - last_modified`
- 如果 `icon_cache_ttl > 0` 且 `icon_cache_ttl <= age.as_secs()` → 过期
- TTL 为 0 表示永不过期；无法获取 mtime 时视为过期

#### 写缓存：`save_icon`

[icons.rs:572-584](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L572-L584)

通过 OpenDAL 的 `operator.write(path, icon).await` 写入。存储后端由配置决定（本地文件系统或 S3 等）。

---

### 3.4 负缓存（失败缓存）机制

#### 检查负缓存：`icon_is_negcached`

[icons.rs:206-228](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L206-L228)

```
1. 检查 {domain}.png.miss 是否过期（使用 icon_cache_negttl）
   ├─ 已过期 → 删除该 .miss 文件 → 返回 false（允许重试下载）
   ├─ 未过期 → 返回 true（继续走降级逻辑）
   └─ .miss 文件不存在/读失败 → 返回 false（允许重试下载）
```

#### 写入负缓存

下载失败（非黑名单错误）时，在 [icons.rs:173-174](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L173-L174) 写入空文件：
```rust
let miss_indicator = path + ".miss";
save_icon(&miss_indicator, vec![]).await;
```

注意：被 `CustomHttpClientError`（黑名单拦截、非全局IP）拦截的请求**不写入**负缓存，因为这属于配置性拒绝而非真实的下载失败，避免错误信息通过缓存"泄露"。

---

### 3.5 HTTP 响应缓存层：`Cached<R>` Responder

[util.rs:210-258](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/util.rs#L210-L258)

无论内部还是外部模式，最终响应都被 `Cached<T>` 包装，它会在 HTTP 响应头设置：

- `Cache-Control: public, max-age=<ttl>` 或 `public, immutable, max-age=<ttl>`
- `Expires: <HTTP-date>`

这是**第二层缓存**（客户端/CDN 缓存），与服务端磁盘缓存并存：

| 场景 | is_immutable | TTL |
|---|---|---|
| 成功返回图标 | `true` | `icon_cache_ttl` |
| 失败返回 fallback | `true` | `icon_cache_negttl` |
| 外部重定向成功 | `true` | `icon_cache_ttl` |
| 外部重定向失败 | `true` | `icon_cache_negttl` |

---

## 四、Favicon 下载链路详解

### 4.1 获取候选图标 URL：`get_icon_url`

[icons.rs:315-401](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L315-L401)

这是一个多级 fallback 的 URL 发现流程：

```
1. 优先尝试 HTTPS://domain
   └─ 非黑名单错误 → 退化为 HTTP://domain
2. HTTP 也失败
   ├─ domain 有 >1 个点且非 IP → 去掉所有子域名，只保留 base.tld，重试 HTTPS → HTTP
   └─ domain 有 <2 个点且非 IP → 加 www. 前缀，重试 HTTPS → HTTP
3. 对最终成功响应的页面：
   ├─ 用响应后的最终 URL（考虑重定向）作为 base
   ├─ 默认追加 /favicon.ico（优先级35）和 /apple-touch-icon.png（优先级40）
   └─ 解析 HTML <head> 中的 <link rel="icon"...> 和 <base href="..."> 标签，
      通过 html5gum 的自定义 FaviconEmitter 提取
4. 页面完全无法访问：
   └─ 直接追加 http(s)://domain/favicon.ico 和 apple-touch-icon.png 共 4 个默认项
5. 按优先级升序排序（数值越小优先级越高）
```

### 4.2 图标优先级计算：`get_icon_priority`

[icons.rs:428-461](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L428-L461)

| 条件 | 优先级值 |
|---|---|
| 32×32 正方形 | 1（最优） |
| 64×64 正方形 | 2 |
| 24-192 之间正方形 | 3 |
| 16×16 正方形 | 4 |
| 其他尺寸正方形 | 5 |
| `.png` 扩展名（无尺寸） | 10 |
| `.jpg`/`.jpeg` 扩展名 | 20 |
| 非正方形（有尺寸） | 200 |
| 默认（其他扩展名） | 30 |
| `.ico` 扩展名回退 | 35 |
| `apple-touch-icon.png` 回退 | 40 |

### 4.3 下载与验证：`download_icon`

[icons.rs:492-570](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L492-L570)

```
按优先级取前 5 个候选图标依次尝试：
  ├─ data:image URI
  │   ├─ 解析失败 → 跳过
  │   ├─ 解码成功但 <67 字节 → 跳过
  │   └─ 魔数校验通过 → 采用
  └─ HTTP(S) URL
      ├─ 请求失败
      │   ├─ 黑名单错误 → 立即终止全部尝试
      │   ├─ 最后一个也失败 → 返回错误
      │   └─ 其他 → 继续下一个
      ├─ 下载（限制 5MB）
      └─ 魔数校验
          ├─ 通过 → 采用
          └─ 不通过 → 清空 buffer，继续下一个

采用后：
  └─ 如果类型是 svg+xml → 用 svg_hush 过滤（防 XSS），过滤失败则丢弃
最终：
  ├─ buffer 非空 → 返回 (bytes, mime_type)
  └─ 全部失败或都被过滤 → 返回错误
```

### 4.4 文件类型识别：`get_icon_type`

[icons.rs:586-611](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L586-L611)

通过文件魔数（magic bytes）识别，支持：
- PNG (`\x89PNG\r\n\x1a\n`)
- ICO (`\x00\x00\x01\x00`)
- WebP (`RIFF....WEBP`)
- JPEG (`\xff\xd8\xff`)
- GIF (`GIF87a`/`GIF89a`)
- BMP (`BM`)
- SVG（`<svg` 或 `<?xml` 开头且首 1KB 内包含 `<svg`）

### 4.5 HTTP 客户端配置

[icons.rs:35-77](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L35-L77)

- 使用全局 `LazyLock<Client>` 复用连接
- 伪装 Chrome 浏览器 User-Agent 和完整请求头（Accept、Accept-Language、Sec-CH-UA、Sec-Fetch-* 等）
- 自定义 Cookie Jar：所有 cookie 强制 2 分钟过期（防止追踪），但保留 cookie 机制以应对需要 cookie 的重定向站点
- 超时：`icon_download_timeout`（默认 10 秒）
- 连接池：每 host 最多 5 个空闲连接，10 秒超时

### 4.6 HTML 解析优化：`FaviconEmitter`

[icons.rs:680-842](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs#L680-L842)

自定义 html5gum Emitter，只解析 `<head>` 范围内的：
- `<link rel="icon" href="..." sizes="...">`
- `<base href="...">`
- `</head>` 结束标签

忽略注释、DOCTYPE、文本内容和无关标签，最小化解析开销。HTML 读取限制为 384KB。

---

## 五、完整协作流程图

```
                    GET /icons/<host>/icon.png
                              │
                              ▼
                  ┌─────────────────────────┐
                  │ ICON_SERVICE = internal?│
                  └─────────┬───────────────┘
                   YES ┌────┴────┐ NO
                       ▼         ▼
              ┌────────────┐  ┌──────────────────┐
              │icon_internal│  │ icon_external    │
              └──────┬─────┘  │ 重定向到外部服务  │
                     │        └────────┬─────────┘
                     ▼                 ▼
         host 校验 + 黑名单校验    host 校验
                     │
        ┌────────────┴────────────┐
        │                         │
     非法/被拦截                  合法
        │                         │
        ▼                         ▼
 返回 fallback.png         ┌───────────────┐
 (Cached, negttl)          │   get_icon()  │
                           └───────┬───────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
           负缓存命中?      正缓存命中?    disable_icon_download?
              │Yes            │Yes             │Yes
              ▼                ▼               ▼
           返回 None       返回缓存         返回 None
              │                │               │
              └──────┬─────────┘               │
                     ▼                         │
           ┌───────────────────┐               │
           │  download_icon()  │               │
           │  多级 URL 回退    │◄──────────────┘
           │  优先级排序        │
           │  data URI 支持    │
           │  魔数校验          │
           │  SVG 安全过滤     │
           └────────┬──────────┘
                    │
           ┌────────┴────────┐
           ▼                 ▼
         成功              失败
           │                 │
           ▼                 ▼
  保存 {domain}.png    ┌──────────────────────┐
  返回图标数据         │ 黑名单错误吗?        │
 (Cached, ttl)        └─────┬──────────────┐
                       Yes ┌┘               └┐ No
                           ▼                  ▼
                      直接返回 None     保存 {domain}.png.miss
                                          返回 None
                                                  │
                                                  ▼
                                    返回 fallback.png
                                     (Cached, negttl)
```

---

## 六、缓存刷新与失效

### 6.1 自然过期

正缓存和负缓存都依赖于磁盘文件的 `last_modified` 时间戳与 TTL 比较，没有独立的清理后台任务。过期检查在每次请求时**惰性**执行。

### 6.2 主动失效

未实现显式的 API 来主动刷新某个图标。要强制刷新，需要：
1. 手动删除 `icon_cache_folder/{domain}.png`（以及可能存在的 `.miss` 文件），或
2. 将 `icon_cache_ttl` 临时设为 0（所有缓存立即过期），请求后再恢复

### 6.3 两级缓存联动

- 服务端磁盘缓存过期 → 重新下载 → 写新缓存 → 新响应带新的 `max-age`
- 客户端缓存先于服务端过期 → 客户端重新请求 → 服务端若磁盘缓存还在则直接返回（带原 `max-age`）
- `is_immutable=true` 提示浏览器/CDN 在 max-age 内不要做条件请求

---

## 七、失败降级总结

| 降级场景 | 返回内容 | HTTP 缓存 TTL | 是否写服务端负缓存 |
|---|---|---|---|
| 域名格式非法 | fallback-icon.png | negttl | 否 |
| 域名匹配黑名单/非全局IP | fallback-icon.png | negttl | 否 |
| 负缓存 .miss 存在且未过期 | fallback-icon.png | negttl | 否（已存在） |
| `disable_icon_download=true` 且无缓存 | fallback-icon.png | negttl | 否 |
| 下载所有候选图标均失败（非黑名单） | fallback-icon.png | negttl | 是（写 .miss） |
| SVG 安全过滤失败 | fallback-icon.png | negttl | 是（写 .miss） |

---

## 八、关键文件速查

| 文件 | 作用 |
|---|---|
| [icons.rs](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/api/icons.rs) | 图标请求处理、缓存读写、favicon 抓取、HTML 解析 |
| [util.rs](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/util.rs#L210-L258) | `Cached<R>` HTTP 响应缓存 Responder、`AppHeaders` Fairing |
| [config.rs](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/config.rs) | 图标相关配置项定义与校验、`PathType::IconCache` 到 OpenDAL Operator 映射 |
| [main.rs](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/main.rs#L590-L590) | `/icons` 路由挂载 |
| [fallback-icon.png](file:///d:/fz/0601/solo-dogfeeding/code/130-vaultwarden/src/static/images/fallback-icon.png) | 内置降级图标（编译时嵌入） |
