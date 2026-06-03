# Vaultwarden 配置层级与默认值来源

## 一、按代码执行顺序的配置加载全景

Vaultwarden 的配置加载严格遵循代码执行顺序，从启动到最终生效分为以下阶段：

```
阶段 A：main() 访问 CONFIG          → 触发 Config::load()
阶段 B：Config::load() → from_env()  → ENV_FILE 定位 → .env 注入 + 环境变量读取
阶段 C：Config::load() → from_file() → 首次访问 CONFIG_FILE LazyLock → 读取 config.json
阶段 D：merge()                      → config.json 覆盖环境变量
阶段 E：build()                      → 用 none_action 填充缺失值
```

核心入口在 [Config::load()](src/config.rs#L1418-L1443)：

```rust
pub async fn load() -> Result<Self, Error> {
    let env = ConfigBuilder::from_env();                        // 阶段 B
    let usr = ConfigBuilder::from_file().await.unwrap_or_default(); // 阶段 C
    let mut overrides = Vec::new();
    let builder = env.merge(&usr, true, &mut overrides);        // 阶段 D
    let config = builder.build();                                // 阶段 E
    // ...
}
```

**最容易混淆的三个点**：
1. `.env` 文件中的变量 **不会** 覆盖外部已存在的同名环境变量（dotenvy::load 行为）
2. `X` 和 `X_FILE` 不是两个覆盖层，而是**同一层内的互斥取值方式**——同时设置会 panic
3. `config.json` 中的值会 **覆盖所有环境变量**（包括外部环境变量），而不是反过来

---

## 二、阶段 A：CONFIG LazyLock 触发 Config::load()

```rust
// src/config.rs#L37-L54
pub static CONFIG: LazyLock<Config> = LazyLock::new(|| {
    std::thread::spawn(|| {
        let rt = tokio::runtime::Builder::new_current_thread()
            .enable_all().build().unwrap_or_else(|e| { exit(12) });
        rt.block_on(Config::load()).unwrap_or_else(|e| { exit(12) })
    }).join().unwrap_or_else(|e| { exit(12) })
});
```

`CONFIG` 是一个 `LazyLock<Config>` 静态变量，**在首次被访问时**才执行初始化闭包。在 [main.rs](src/main.rs#L79) 中，`init_logging()` 调用 `CONFIG.log_level()` 时首次触发 `CONFIG` 的初始化，从而调用 `Config::load()`。

**关键点**：`Config::load()` 的执行时机是在 `main()` 函数的早期，此时只有外部环境变量（Docker env / Shell export / systemd Environment）存在于进程环境中，.env 文件尚未加载。

---

## 三、阶段 B：from_env() —— 环境变量层的完整流程

[from_env()](src/config.rs#L218-L260) 是环境变量层的核心，内部按严格顺序执行三个步骤：

### 步骤 B-1：ENV_FILE 引导 .env 路径

```rust
// src/config.rs#L219
let env_file = get_env("ENV_FILE").unwrap_or_else(|| String::from(".env"));
```

此时 .env 文件尚未加载，`get_env("ENV_FILE")` 只能从**外部环境变量**中读取。

`ENV_FILE` 的作用是**告诉 dotenvy 去哪里找 .env 文件**，默认值为当前目录下的 `.env`。

⚠️ **`ENV_FILE` 无法通过 .env 文件自身设置**——因为读取 `ENV_FILE` 在加载 .env 之前。这是一个"鸡生蛋"的问题：你需要先知道 .env 文件在哪里，才能加载它。

### 步骤 B-2：dotenvy::from_path() —— .env 注入，外部变量保留

```rust
// src/config.rs#L220
match dotenvy::from_path(&env_file) { ... }
```

Vaultwarden 使用 `dotenvy` crate 0.15.7 的 `from_path()` 函数。该函数内部调用 `Iter::load()`，其核心逻辑是：

```rust
// dotenvy iter.rs 源码
pub fn load(mut self) -> Result<()> {
    for item in self {
        let (key, value) = item?;
        if env::var(&key).is_err() {   // ← 只在变量不存在时才设置
            env::set_var(&key, value);
        }
    }
    Ok(())
}
```

**这意味着**：.env 文件中的变量 **不会** 覆盖进程环境中已存在的同名变量。外部环境变量（如 Docker `--env`、Shell `export`、systemd `Environment=`）始终优先于 .env 文件中的同名变量。

dotenvy 还提供了 `load_override()` 方法（无条件覆盖），但 Vaultwarden **没有使用**它。

**dotenvy::from_path() 的错误处理**：

| 错误类型 | 是否显式设置了 ENV_FILE？ | 行为 | exit code |
|---|---|---|---|
| 文件格式解析错误 | 任意 | 打印错误 + 退出 | 255 |
| 文件不存在（NotFound） | 是 | 打印错误 + 退出 | 255 |
| 文件不存在（NotFound） | 否（使用默认 `.env`） | **静默跳过**，不退出 | - |
| 权限不足（PermissionDenied） | 任意 | 打印错误 + 退出 | 255 |
| 其他 IO 错误 | 任意 | 打印错误 + 退出 | 255 |
| 其他 dotenvy 错误 | 任意 | 打印错误 + 退出 | 255 |

**设计意图**：默认 `.env` 不存在是正常情况（生产环境通常用外部环境变量），但显式指定了 `ENV_FILE` 却找不到则说明配置有误，必须报错退出。

### 步骤 B-3：逐项读取环境变量 —— X 与 X_FILE 互斥

.env 注入完成后，`from_env()` 遍历所有配置项，通过 `make_config!` 宏展开为：

```rust
// src/config.rs#L254-L257
builder.$name = make_config! { @getenv stringify!([<$name:upper>]), $ty };
```

宏展开后实际调用 [get_env()](src/util.rs#L412-L417) 或 [get_env_bool()](src/util.rs#L419-L428)，两者都依赖 [get_env_str_value()](src/util.rs#L396-L410) 获取原始字符串：

```rust
pub fn get_env_str_value(key: &str) -> Option<String> {
    let key_file = format!("{key}_FILE");
    let value_from_env = env::var(key);        // 读取 X
    let value_file = env::var(&key_file);      // 读取 X_FILE

    match (value_from_env, value_file) {
        (Ok(_), Ok(_)) => panic!("You should not define both {key} and {key_file}!"),
        (Ok(v_env), Err(_)) => Some(v_env),
        (Err(_), Ok(v_file)) => match std::fs::read_to_string(v_file) {
            Ok(content) => Some(content.trim().to_owned()),
            Err(e) => panic!("Failed to load {key}: {e:?}"),
        },
        _ => None,
    }
}
```

**X 与 X_FILE 不是两个覆盖层，而是同一配置项的两种互斥取值方式**：

| X 环境变量 | X_FILE 环境变量 | 行为 | 结果 |
|---|---|---|---|
| 设置 | 未设置 | 使用 X 的值 | `Some(X的值)` |
| 未设置 | 设置 | 从文件读取（自动 trim） | `Some(文件内容)` |
| **设置** | **设置** | **panic！** 程序直接崩溃 | - |
| 未设置 | 未设置 | 返回 None | `None` |
| 未设置 | 设置但文件读失败 | **panic！** 程序直接崩溃 | - |

`X_FILE` 的设计目的是支持 Docker Secrets / Kubernetes Secrets 等场景，其中敏感值以文件形式挂载（如 `/run/secrets/admin_token`）。

**获取到字符串后的类型转换**：

- [get_env()](src/util.rs#L412-L417)：通过 [try_parse_string()](src/util.rs#L381-L391) 调用 `FromStr::parse()`，解析失败返回 `None`（静默回退到默认值，不报错）
- [get_env_bool()](src/util.rs#L419-L428)：布尔专用，支持 `true/t/yes/y/1` 和 `false/f/no/n/0`（不区分大小写），其他值返回 `None`

### 步骤 B 小结：环境变量层内部的优先级

```
外部环境变量（Shell/Docker/systemd 注入）
          ↓ 优先于
.env 文件中的同名变量（dotenvy::load 不覆盖）
          ↓ 取值方式
X 直接值 或 X_FILE 文件值（两者互斥，冲突 panic）
          ↓ 类型转换
成功解析为对应类型 → 写入 ConfigBuilder 的 Some(value)
解析失败或未设置  → 保持 None（留给 build() 阶段填充默认值）
```

---

## 四、阶段 C：from_file() —— CONFIG_FILE LazyLock 首次触发 + 读取 config.json

这是理解配置加载顺序的关键阶段。[from_file()](src/config.rs#L262-L267) 的代码：

```rust
async fn from_file() -> Result<Self, Error> {
    let operator = storage::operator_for_path(&CONFIG_FILE_PARENT_DIR)?;  // ← 首次访问 CONFIG_FILE
    let config_bytes = operator.read(&CONFIG_FILENAME).await?;
    println!("[INFO] Using saved config from `{}` for configuration.\n", *CONFIG_FILE);
    serde_json::from_slice(&config_bytes.to_vec()).map_err(Into::into)
}
```

### 4.1 CONFIG_FILE LazyLock 的真实触发时机

`CONFIG_FILE` 是 `LazyLock<String>`，声明在 [config.rs#L24-L27](src/config.rs#L24-L27)：

```rust
static CONFIG_FILE: LazyLock<String> = LazyLock::new(|| {
    let data_folder = get_env("DATA_FOLDER").unwrap_or_else(|| String::from("data"));
    get_env("CONFIG_FILE").unwrap_or_else(|| storage::join_path(&data_folder, "config.json"))
});
```

LazyLock **不会在声明时执行**，只在**首次访问时执行**初始化闭包。通过搜索全部代码，`CONFIG_FILE`（及其依赖的 `CONFIG_FILE_PARENT_DIR`、`CONFIG_FILENAME`）仅在以下位置被访问：

| 访问位置 | 时机 |
|---|---|
| [from_file()](src/config.rs#L263-L265) | `Config::load()` 内，`from_env()` 之后 |
| [update_config()](src/config.rs#L1477-L1478) | 运行时 Admin 面板更新 |
| [delete_config()](src/config.rs#L1542-L1543) | 运行时 Admin 面板删除 |

**首次触发一定是在 `from_file()` 中**，而 `from_file()` 在 `from_env()` 之后执行。

### 4.2 这意味着什么：DATA_FOLDER 和 CONFIG_FILE 可以从 .env 设置

由于 `CONFIG_FILE` LazyLock 在 `from_file()` 中才首次被访问，此时 `from_env()` **已经执行完毕**，.env 文件中的变量已经被注入到进程环境中。

因此 `CONFIG_FILE` 初始化闭包中的 `get_env("DATA_FOLDER")` 和 `get_env("CONFIG_FILE")` **可以读取到 .env 文件中设置的值**。

完整路径解析规则：

```
CONFIG_FILE 环境变量设置了？（来自外部环境变量 或 .env 文件）
  → 是 → 使用 CONFIG_FILE 的值
  → 否 → DATA_FOLDER 环境变量设置了？（来自外部环境变量 或 .env 文件）
           → 是 → DATA_FOLDER + "/config.json"
           → 否 → "data/config.json"
```

### 4.3 三个"引导变量"的 .env 可用性对比

| 变量 | 读取时机 | .env 中设置是否生效 | 原因 |
|---|---|---|---|
| `ENV_FILE` | `from_env()` 步骤 B-1 | ❌ 不生效 | 读取时 .env 尚未加载（鸡生蛋问题） |
| `DATA_FOLDER` | `from_file()` 阶段 C 的 CONFIG_FILE LazyLock | ✅ 生效 | .env 已在阶段 B 加载 |
| `CONFIG_FILE` | `from_file()` 阶段 C 的 CONFIG_FILE LazyLock | ✅ 生效 | .env 已在阶段 B 加载 |

### 4.4 config.json 的加载与反序列化

`from_file()` 读取由 CONFIG_FILE 确定的路径。

**自定义反序列化器**（[config.rs#L120-L207](src/config.rs#L120-L207)）：Vaultwarden 为 `ConfigBuilder` 实现了自定义的 `Deserialize`，特点：
- **忽略未知字段**：JSON 中的多余键不会导致反序列化失败
- **检测重复键**：同一键出现两次会报错
- **所有字段可选**：缺失的字段保持 `None`

### 4.5 from_file 失败场景与 unwrap_or_default 回退机制

`from_file()` 是一个 `Result<Self, Error>`，在 `Config::load()` 中被 `.unwrap_or_default()` 处理：

```rust
// src/config.rs#L1421
let usr = ConfigBuilder::from_file().await.unwrap_or_default();
```

**这意味着：任何失败都会静默回退到 `ConfigBuilder::default()`，即所有字段为 `None` 的空配置。**

#### 4.5.1 from_file 可能失败的四个位置

| 失败位置 | 错误来源 | 触发条件 | 结果 |
|---|---|---|---|
| `operator_for_path()` | 存储后端初始化失败 | 路径无效、权限不足等 | 回退空配置 |
| `operator.read()` | 文件读取失败 | 文件不存在、权限不足、磁盘 IO 错误 | 回退空配置 |
| `serde_json::from_slice()` | JSON 语法错误 | 格式非法（如括号不匹配）、类型不匹配 | 回退空配置 |
| 自定义反序列化器 | 重复键检测 | JSON 中同一键出现两次 | 回退空配置 |

**关键结论**：无论是文件不存在、JSON 语法错误、还是重复键，都不会阻止程序启动——只是 config.json 层的所有配置丢失，完全回退到环境变量层。

#### 4.5.2 JSON 重复键的处理：代码 vs 注释的差异

注释第 118 行声称：
> "In case of duplicate keys ... the last value is used!"

但实际代码在第 190-192 行明确检查了重复键并返回错误：

```rust
// src/config.rs#L189-L194
Field::$name => {
    if builder.$name.is_some() {
        return Err(de::Error::duplicate_field(stringify!($name)));
    }
    builder.$name = map.next_value()?;
}
```

**实际行为：重复键 → 反序列化失败 → unwrap_or_default → 回退空配置。**

注释是错误的，应以实际代码为准。

#### 4.5.3 回退空配置对 merge 覆盖关系的影响

merge() 的核心逻辑是：

```rust
// src/config.rs#L282-L283
if let v @ Some(_) = &other.$name {
    builder.$name = v.clone();
}
```

只有 `other`（config.json 来源）中为 `Some(_)` 的字段才会覆盖。如果 `from_file()` 失败并回退到 `ConfigBuilder::default()`，则所有字段都是 `None`，**不会覆盖任何环境变量**。

**覆盖行为对照表**：

| config.json 状态 | usr 中字段值 | merge 时是否覆盖环境变量 |
|---|---|---|
| 文件存在且合法，某字段有值 | `Some(value)` | ✅ 覆盖，打印警告（如果环境变量也设置了） |
| 文件存在且合法，某字段无值 | `None` | ❌ 不覆盖，保留环境变量值 |
| 文件不存在 / 读取失败 / 解析失败 | 全 `None` | ❌ 完全不覆盖，所有配置来自环境变量 |

#### 4.5.4 失败时的用户感知

- **没有错误日志**：`.unwrap_or_default()` 静默吞掉所有错误
- **没有警告提示**：文件损坏或被删除时，管理员可能完全不知情
- **效果等同于清空 Admin 面板的所有设置**：所有曾经保存到 config.json 的配置瞬间消失
- **静默回退可能导致意外行为**：比如 `DOMAIN` 突然从 `https://prod.example.com` 回退到环境变量的 `https://localhost`，导致回调 URL 失效

---

## 五、阶段 D：merge() —— config.json 覆盖环境变量

[merge()](src/config.rs#L279-L299) 将环境变量层（env）和 config.json 层（usr）合并：

```rust
/// Merges the values of both builders into a new builder.
/// If both have the same element, `other` wins.
fn merge(&self, other: &Self, show_overrides: bool, overrides: &mut Vec<&str>) -> Self {
    let mut builder = self.clone();
    // other（config.json 来源）中某字段为 Some → 覆盖 self（环境变量来源）
    // 如果 self 中也有值 → 记录到 overrides 列表
}
```

**关键规则**：`other`（config.json）中为 `Some(_)` 的字段会覆盖 `self`（环境变量）中的值。如果环境变量中也设置了同名变量，会打印警告：

> `[WARNING] The following environment variables are being overridden by the config.json file.`

---

## 六、阶段 E：build() —— 用 none_action 填充缺失值

[build()](src/config.rs#L301-L321) 将 `ConfigBuilder`（所有字段为 `Option`）转换为 `ConfigItems`（字段有具体类型）。

`make_config!` 宏中定义了四种 `none_action`：

| none_action | 含义 | build 时的行为 | 类型签名 |
|---|---|---|---|
| `def` | 静态默认值 | `value.unwrap_or(default)` | `T` |
| `auto` | 动态计算 | `Some(v) → v`，`None → f(&config)` | `T` |
| `option` | 可选，无默认值 | 直接使用（可能是 `None`） | `Option<T>` |
| `generated` | 始终自动生成 | 忽略用户值，总是 `f(&config)` | `T` |

典型示例：

```rust
data_folder:        String, false, def,       "data".to_owned();
database_url:       String, false, auto,      |c| format!("sqlite://{}", ...);
hibp_api_key:       Pass,   true,  option;
_ip_header_enabled: bool,   false, generated, |c| &c.ip_header.trim().to_lowercase() != "none";
```

额外后处理：`domain` 去 `/`、`signups_domains_whitelist`/`org_creation_users` trim+小写、`icon_blacklist_regex` → `http_request_block_regex` 兼容迁移。

---

## 七、`make_config!` 宏 —— 声明式配置定义

所有配置项在一个大型宏调用中声明，位于 [config.rs#L502-L921](src/config.rs#L502-L921)。每个配置项的格式为：

```
/// Friendly Name |> Description
name: type, is_editable, none_action, default_value?;
```

### 7.1 类型系统

| 宏中类型 | Rust 实际类型 | 环境变量读取 | Admin 表单类型 |
|---|---|---|---|
| `String` | `String` | `get_env()` | `text` |
| `Pass` | `String`（密码） | `get_env()` | `password` |
| `bool` | `bool` | `get_env_bool()` | `checkbox` |
| `u16`/`u32`/`u64`/`i32`/`i64`/`u8`/`usize` | 对应数值类型 | `get_env()` → `FromStr` | `number` |

`Pass` 类型在序列化展示时会被掩码为 `"***"`，用于密码、密钥等敏感字段。

### 7.2 is_editable 标记

`is_editable` 控制该配置项是否可以在 Admin 面板中修改。`false` 表示只能通过环境变量设置，见 [clear_non_editable()](src/config.rs#L269-L275)。

---

## 八、运行时更新

当通过 Admin 面板更新配置时（[update_config()](src/config.rs#L1445-L1481)）：

1. 清除不可编辑字段（`clear_non_editable()`）
2. 将新配置与原始 `env` 合并（`env.merge(&builder, ...)`）—— config.json 仍然覆盖环境变量
3. 校验（比启动时更严格，某些警告场景会变成错误）
4. 写入内存 + 写入 config.json 文件

---

## 九、配置校验

[validate_config()](src/config.rs#L923-L1267) 在 `build()` 之后执行，包含大量业务规则校验，例如：
- `PASSWORD_ITERATIONS` 必须 ≥ 100000
- `DOMAIN` 必须以 `http://` 或 `https://` 开头
- `PUSH_ENABLED` 时必须提供 `PUSH_INSTALLATION_ID` 和 `PUSH_INSTALLATION_KEY`
- SMTP 相关字段间的一致性检查
- cron 表达式格式校验
- `ADMIN_TOKEN` 格式校验（Argon2 PHC 或明文警告）

校验失败会阻止启动。运行时更新时的校验更严格，某些仅打印警告的场景在更新时会直接报错。

---

## 十、配置优先级总览

以 `DOMAIN` 配置项为例，按代码执行顺序梳理的完整优先级链：

```
阶段 E: build() 默认值         "http://localhost"       ← 最低优先级
          ↑ 被覆盖
阶段 B: .env 文件              DOMAIN=https://env.example.com  ← 只有外部未设置时生效
          ↑ 被覆盖（dotenvy::load 不覆盖外部变量）
阶段 B: 外部环境变量            DOMAIN=https://docker.example.com ← Shell/Docker/systemd 注入
          ↑ 被覆盖
          │ （X 和 X_FILE 是同一层的互斥取值方式，不是两个覆盖层）
阶段 D: config.json            {"domain": "https://admin.example.com"} ← Admin 面板写入
          ↑ 被覆盖
运行时:  Admin 面板即时修改     → 写入 config.json → 重新合并
```

**实际生效规则（重点）**：
1. `.env` 文件中的变量 **不会** 覆盖外部已存在的同名环境变量
2. `X` 与 `X_FILE` 是同一配置项的两种互斥取值方式，同时设置 → **panic**
3. `config.json` 中的值覆盖所有环境变量（包括外部环境变量和 .env 文件）
4. `config.json` 读取失败（文件不存在 / JSON 语法错误 / 重复键等）→ **静默回退到空配置**，所有字段为 `None`，不覆盖任何环境变量
5. 未在任何层级设置的值，由 `none_action` 策略决定最终值
6. `ENV_FILE` 只能通过外部环境变量设置（.env 文件中设置无效，因为读取时 .env 尚未加载）
7. `DATA_FOLDER` 和 `CONFIG_FILE` **可以** 通过 .env 文件设置（因为 CONFIG_FILE LazyLock 在 .env 加载之后才首次被访问）

---

## 十一、关键代码位置索引

| 概念 | 位置 |
|---|---|
| `CONFIG_FILE` LazyLock 声明 | [config.rs#L24-L27](src/config.rs#L24-L27) |
| `CONFIG` LazyLock 声明 | [config.rs#L37-L54](src/config.rs#L37-L54) |
| `make_config!` 宏定义 | [config.rs#L58-L488](src/config.rs#L58-L488) |
| 所有配置项声明 | [config.rs#L502-L921](src/config.rs#L502-L921) |
| `Config::load()` 入口 | [config.rs#L1418-L1443](src/config.rs#L1418-L1443) |
| `from_env()` 环境变量构建 | [config.rs#L218-L260](src/config.rs#L218-L260) |
| ENV_FILE 引导 .env 路径 | [config.rs#L219](src/config.rs#L219) |
| dotenvy::from_path 调用 | [config.rs#L220](src/config.rs#L220) |
| ENV_FILE 错误处理分支 | [config.rs#L224-L251](src/config.rs#L224-L251) |
| 逐项读取环境变量（宏展开） | [config.rs#L254-L257](src/config.rs#L254-L257) |
| `from_file()` config.json 加载 | [config.rs#L262-L267](src/config.rs#L262-L267) |
| `unwrap_or_default()` 静默回退 | [config.rs#L1421](src/config.rs#L1421) |
| 自定义反序列化器（含重复键检测） | [config.rs#L120-L207](src/config.rs#L120-L207) |
| 重复键检测代码 | [config.rs#L190-L192](src/config.rs#L190-L192) |
| `merge()` 合并覆盖 | [config.rs#L279-L299](src/config.rs#L279-L299) |
| merge 条件覆盖逻辑 | [config.rs#L282-L283](src/config.rs#L282-L283) |
| `build()` 默认值填充 | [config.rs#L301-L321](src/config.rs#L301-L321) |
| `update_config()` 运行时更新 | [config.rs#L1445-L1481](src/config.rs#L1445-L1481) |
| `get_env_str_value()` 含 X/X_FILE 互斥 | [util.rs#L396-L410](src/util.rs#L396-L410) |
| `get_env()` 泛型读取 | [util.rs#L412-L417](src/util.rs#L412-L417) |
| `get_env_bool()` 布尔读取 | [util.rs#L419-L428](src/util.rs#L419-L428) |
| `try_parse_string()` 类型转换 | [util.rs#L381-L391](src/util.rs#L381-L391) |
| `validate_config()` 校验 | [config.rs#L923-L1267](src/config.rs#L923-L1267) |
| main() 中首次访问 CONFIG | [main.rs#L79](src/main.rs#L79) |
| `.env.template` 模板 | [.env.template](.env.template) |

**外部库参考**：
- `dotenvy::Iter::load()` 非覆盖实现：[dotenvy 0.15.7 源码](https://docs.rs/dotenvy/0.15.7/src/dotenvy/iter.rs.html#29-L40)
- `dotenvy::Iter::load_override()` 覆盖实现（Vaultwarden 未使用）：[dotenvy 0.15.7 源码](https://docs.rs/dotenvy/0.15.7/src/dotenvy/iter.rs.html#47-L56)
