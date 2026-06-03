# Vaultwarden 配置层级与默认值来源

## 一、配置来源总览

Vaultwarden 的配置值来自 **四个层级**，优先级从低到高：

```
硬编码默认值 (def/auto/generated)
    ↑ 被覆盖
环境变量 / .env 文件  (env)
    ↑ 被覆盖
config.json 配置文件  (usr)
    ↑ 被覆盖
运行时 Admin 面板修改  (也写入 config.json)
```

核心代码位于 [`src/config.rs`](src/config.rs) 的 `Config::load()` 方法（第 1418-1443 行）：

```rust
let env = ConfigBuilder::from_env();
let usr = ConfigBuilder::from_file().await.unwrap_or_default();
let mut overrides = Vec::new();
let builder = env.merge(&usr, true, &mut overrides);
let config = builder.build();
```

**关键结论**：config.json 的值 **覆盖** 环境变量（而非反过来），这在 [merge](src/config.rs#L279-L299) 方法中有明确注释：

> Merges the values of both builders into a new builder. If both have the same element, `other` wins.

---

## 二、环境变量层

### 2.1 加载流程

1. **加载 .env 文件**：通过 `dotenvy::from_path()` 读取 `.env` 文件（路径由 `ENV_FILE` 环境变量指定，默认为 `.env`），将文件中的键值对注入到进程环境中。见 [from_env()](src/config.rs#L218-L260)。

2. **逐项读取环境变量**：通过 `make_config!` 宏展开，对每个配置项调用 `get_env` 或 `get_env_bool` 读取对应的大写环境变量名。例如 `data_folder` 会读取 `DATA_FOLDER`。

### 2.2 环境变量读取机制

代码位于 [`src/util.rs`](src/util.rs#L396-L428)：

```
get_env_str_value(key)          ← 原始字符串获取
    ├── 读取 ENV_VAR           ← std::env::var(key)
    ├── 读取 ENV_VAR_FILE      ← std::env::var(key + "_FILE")，从文件中读取值
    └── 两者不可同时设置，否则 panic

get_env<V>(key)                 ← 泛型版本，调用 try_parse_string 做类型转换
get_env_bool(key)               ← 布尔专用版本，支持 true/t/yes/y/1 和 false/f/no/n/0
```

**特殊机制：`_FILE` 后缀**

每个环境变量 `X` 都可以用 `X_FILE` 替代，`X_FILE` 的值是一个文件路径，Vaultwarden 会从该文件读取内容作为配置值（自动 trim）。例如：
- `ADMIN_TOKEN=xxx` 等价于 `ADMIN_TOKEN_FILE=/path/to/secret_file`
- 两者**不可同时设置**，否则程序直接 panic

### 2.3 布尔类型转换规则

[get_env_bool](src/util.rs#L419-L428) 的转换表：

| 值（不区分大小写） | 结果 |
|---|---|
| `true`, `t`, `yes`, `y`, `1` | `true` |
| `false`, `f`, `no`, `n`, `0` | `false` |
| 其他 / 未设置 | `None`（使用默认值） |

### 2.4 数值/字符串类型转换

[try_parse_string](src/util.rs#L381-L391) 使用 Rust 标准库的 `FromStr` trait 进行转换。如果解析失败（如将 `abc` 赋给 `u64` 类型的配置项），返回 `None`，该配置项将回退到默认值——**不会报错**。

---

## 三、配置文件层（config.json）

### 3.1 文件路径

由 [`CONFIG_FILE`](src/config.rs#L24-L27) 决定：

```
CONFIG_FILE 环境变量 → 若设置了则使用该值
                  → 否则 = DATA_FOLDER + "/config.json"
                  → DATA_FOLDER 默认 = "data"
```

默认路径为 `data/config.json`。

### 3.2 加载流程

[from_file()](src/config.rs#L262-L267) 使用 `serde_json` 反序列化 JSON 文件为 `ConfigBuilder`。

**自定义反序列化器**（第 120-207 行）：Vaultwarden 为 `ConfigBuilder` 实现了自定义的 `Deserialize`，特点：
- **忽略未知字段**：JSON 中的多余键不会导致反序列化失败
- **检测重复键**：同一键重复且前一个值已经解析为 `Some` 时会报错；未知字段会被忽略
- **所有字段可选**：缺失的字段保持 `None`

### 3.3 config.json 的来源

config.json 主要由 **Admin 面板** 产生和修改。当管理员在 Web 界面修改配置时，调用 [update_config()](src/config.rs#L1445-L1481)，将 `ConfigBuilder` 序列化为 JSON 写入文件。

---

## 四、`make_config!` 宏 —— 声明式配置定义

所有配置项在一个大型宏调用中声明，位于 [config.rs 第 502-921 行](src/config.rs#L502-L921)。每个配置项的格式为：

```
/// Friendly Name |> Description
name: type, is_editable, none_action, default_value?;
```

### 4.1 四种 none_action（缺省策略）

| none_action | 含义 | 默认值来源 | 类型签名 |
|---|---|---|---|
| `def` | 使用静态默认值 | 宏声明中直接指定 | `T` |
| `auto` | 基于其他配置项动态计算 | 闭包 `\|c\| ...` | `T` |
| `option` | 可选，无默认值 | 无，值为 `None` | `Option<T>` |
| `generated` | 始终自动生成，忽略用户输入 | 闭包 `\|c\| ...` | `T` |

**`@build` 展开逻辑**（第 75-89 行）：

```rust
// option: 直接使用值（可能是 None）
( @build $value:expr, $config:expr, option, ) => { $value };

// def: 值为 None 时使用默认值
( @build $value:expr, $config:expr, def, $default:expr ) => {
    $value.unwrap_or($default)
};

// auto: 值为 None 时调用闭包计算
( @build $value:expr, $config:expr, auto, $default_fn:expr ) => {{
    match $value {
        Some(v) => v,
        None => { let f: &dyn Fn(&ConfigItems) -> _ = &$default_fn; f($config) }
    }
}};

// generated: 始终忽略用户值，总是调用闭包计算
( @build $value:expr, $config:expr, generated, $default_fn:expr ) => {{
    let f: &dyn Fn(&ConfigItems) -> _ = &$default_fn; f($config)
}};
```

### 4.2 典型示例

```rust
// def —— 静态默认值
data_folder:            String, false, def,    "data".to_owned();

// auto —— 动态计算，依赖其他配置项
database_url:           String, false, auto, |c| format!("sqlite://{}", storage::join_path(&c.data_folder, "db.sqlite3"));

// option —— 可选配置，无默认值
hibp_api_key:           Pass,   true,  option;

// generated —— 始终生成，用户输入无效
_ip_header_enabled:     bool,   false, generated, |c| &c.ip_header.trim().to_lowercase() != "none";
```

### 4.3 类型系统

宏中支持的类型及对应的环境变量读取方式：

| 宏中类型 | Rust 实际类型 | 环境变量读取 | Admin 表单类型 |
|---|---|---|---|
| `String` | `String` | `get_env()` | `text` |
| `Pass` | `String`（密码） | `get_env()` | `password` |
| `bool` | `bool` | `get_env_bool()` | `checkbox` |
| `u16`/`u32`/`u64`/`i32`/`i64`/`u8`/`usize` | 对应数值类型 | `get_env()` → `FromStr` | `number` |

`Pass` 类型在序列化展示时会被掩码为 `"***"`，用于密码、密钥等敏感字段。

### 4.4 is_editable 标记

第二个字段 `is_editable` 控制该配置项是否可以在 Admin 面板中修改。`false` 表示不可通过 Admin 面板修改（只能通过环境变量设置），见 [clear_non_editable()](src/config.rs#L269-L275)。

---

## 五、合并与覆盖逻辑

### 5.1 启动时合并

[Config::load()](src/config.rs#L1418-L1443) 的流程：

```
1. env  = ConfigBuilder::from_env()          // 从环境变量构建
2. usr  = ConfigBuilder::from_file()          // 从 config.json 构建（失败则全为 None）
3. builder = env.merge(&usr, ...)             // 合并：usr（config.json）覆盖 env
4. config = builder.build()                   // 用 none_action 填充缺失值
5. validate_config(&config, false)            // 校验
```

### 5.2 merge 方法

[merge()](src/config.rs#L279-L299) 的行为：

- 遍历所有配置项
- 如果 `other`（config.json 来源）中某字段为 `Some(_)`，则覆盖 `self`（环境变量来源）中的值
- 如果 `self` 中也有值（即环境变量已设置），则记录到 `overrides` 列表，并打印警告：

> `[WARNING] The following environment variables are being overridden by the config.json file.`

### 5.3 build 方法

[build()](src/config.rs#L301-L321) 根据 `none_action` 将 `ConfigBuilder`（所有字段为 `Option`）转换为 `ConfigItems`（字段有具体类型）。额外后处理：
- `domain` 末尾 `/` 被去除
- `signups_domains_whitelist` 被 trim 并转小写
- `org_creation_users` 被 trim 并转小写
- 废弃字段 `icon_blacklist_regex` → `http_request_block_regex` 的兼容迁移

### 5.4 运行时更新

当通过 Admin 面板更新配置时（[update_config()](src/config.rs#L1445-L1481)）：

1. 清除不可编辑字段（`clear_non_editable()`）
2. 将新配置与原始 `env` 合并（`env.merge(&builder, ...)`）
3. 校验
4. 写入内存 + 写入 config.json 文件

---

## 六、配置优先级总结

以 `DOMAIN` 配置项为例，完整的优先级链：

```
1. 硬编码默认值:    "http://localhost"       ← make_config! 中 def 指定
2. 环境变量:        DOMAIN=https://vw.example.com   ← .env 文件或容器环境变量注入
3. _FILE 后缀:      DOMAIN_FILE=/run/secrets/domain ← 从文件读取（与环境变量互斥）
4. config.json:     {"domain": "https://prod.example.com"}  ← Admin 面板写入
5. 运行时更新:      Admin 面板即时修改 → 写入 config.json → 重新合并
```

**实际生效规则**：
- 环境变量与 `_FILE` 互斥，不可同时设置
- config.json 中的值覆盖环境变量中的同名字段
- 未在任何层级设置的值，由 `none_action` 策略决定最终值

---

## 七、.env 文件加载细节

[from_env()](src/config.rs#L218-L260) 中 `.env` 文件的处理：

| 情况 | 行为 |
|---|---|
| `.env` 文件存在且合法 | 打印 `[INFO] Using environment file ...`，加载变量 |
| `.env` 文件不存在，且未设置 `ENV_FILE` | 静默跳过（正常情况） |
| 设置了 `ENV_FILE` 但文件不存在 | 打印错误并 `exit(255)` |
| `.env` 文件权限不足 | 打印错误并 `exit(255)` |
| `.env` 文件格式错误 | 打印错误（含行号和位置）并 `exit(255)` |

---

## 八、配置校验

[validate_config()](src/config.rs#L923-L1267) 在 `build()` 之后执行，包含大量业务规则校验，例如：
- `PASSWORD_ITERATIONS` 必须 ≥ 100000
- `DOMAIN` 必须以 `http://` 或 `https://` 开头
- `PUSH_ENABLED` 时必须提供 `PUSH_INSTALLATION_ID` 和 `PUSH_INSTALLATION_KEY`
- SMTP 相关字段间的一致性检查
- cron 表达式格式校验
- `ADMIN_TOKEN` 格式校验（Argon2 PHC 或明文警告）

校验失败会阻止启动。但运行时更新（Admin 面板）时的校验更严格，某些仅打印警告的场景在更新时会直接报错。

---

## 九、关键代码位置索引

| 概念 | 文件 | 行号 |
|---|---|---|
| `make_config!` 宏定义 | [`src/config.rs`](src/config.rs#L58-L488) | 58-488 |
| 所有配置项声明 | [`src/config.rs`](src/config.rs#L502-L921) | 502-921 |
| `Config::load()` 入口 | [`src/config.rs`](src/config.rs#L1418-L1443) | 1418-1443 |
| `from_env()` 环境变量构建 | [`src/config.rs`](src/config.rs#L218-L260) | 218-260 |
| `from_file()` 配置文件加载 | [`src/config.rs`](src/config.rs#L262-L267) | 262-267 |
| `merge()` 合并覆盖 | [`src/config.rs`](src/config.rs#L279-L299) | 279-299 |
| `build()` 默认值填充 | [`src/config.rs`](src/config.rs#L301-L321) | 301-321 |
| `update_config()` 运行时更新 | [`src/config.rs`](src/config.rs#L1445-L1481) | 1445-1481 |
| `get_env_str_value()` 原始读取 | [`src/util.rs`](src/util.rs#L396-L410) | 396-410 |
| `get_env()` 泛型读取 | [`src/util.rs`](src/util.rs#L412-L417) | 412-417 |
| `get_env_bool()` 布尔读取 | [`src/util.rs`](src/util.rs#L419-L428) | 419-428 |
| `try_parse_string()` 类型转换 | [`src/util.rs`](src/util.rs#L381-L391) | 381-391 |
| `validate_config()` 校验 | [`src/config.rs`](src/config.rs#L923-L1267) | 923-1267 |
| `.env.template` 模板 | [`.env.template`](.env.template) | - |
| Admin 面板配置写入 | [`src/api/admin.rs`](src/api/admin.rs#L798-L804) | 798-804 |
