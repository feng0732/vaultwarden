# 多数据库迁移链路分析

## 一、整体架构概览

Vaultwarden 使用 Diesel ORM 框架实现多数据库支持，通过**编译时特性标记**和**独立迁移目录**的方式，实现 SQLite、PostgreSQL、MySQL 三种数据库的兼容。

### 1.1 核心组件位置

| 组件 | 文件路径 |
|------|----------|
| 数据库连接管理 | `src/db/mod.rs` |
| SQLite 迁移目录 | `migrations/sqlite/` |
| MySQL 迁移目录 | `migrations/mysql/` |
| PostgreSQL 迁移目录 | `migrations/postgresql/` |
| 编译特性配置 | `Cargo.toml` |
| 构建脚本 | `build.rs` |
| 数据库 Schema 定义 | `src/db/schema.rs` |

---

## 二、迁移组织方式

### 2.1 目录结构与脚本数量（已核准）

```
migrations/
├── sqlite/          # 56 个迁移脚本 (2018-01-14 ~ 2026-05-05)
│   ├── 2018-01-14-171611_create_tables/
│   ├── ...
│   └── 2026-05-05-120000_sso_auth_error/
├── mysql/           # 55 个迁移脚本 (2018-01-14 ~ 2026-05-05)
│   ├── 2018-01-14-171611_create_tables/
│   ├── ...
│   └── 2026-05-05-120000_sso_auth_error/
└── postgresql/      # 46 个迁移脚本 (2019-09-12 ~ 2026-05-05)
    ├── 2019-09-12-100000_create_tables/
    ├── ...
    └── 2026-05-05-120000_sso_auth_error/
```

**数量差异原因（基于目录集合精确对比）**：

---

#### SQLite vs MySQL：56 vs 55，净差 1 个

**SQLite 有而 MySQL 没有的（3个）**：
1. `2021-03-15-163412_rename_send_key` — MySQL 在 `2021-03-11_add_sends` 创建 sends 表时直接用 `akey`，无需改名
2. `2024-02-14-140000_change_time_stamp_data_type` — 同一功能但时间戳不同
3. `2024-03-13_170000_sso_userscascade` — 同一功能但分隔符/命名不同

**MySQL 有而 SQLite 没有的（2个）**：
1. `2024-02-14-135828_change_time_stamp_data_type`
2. `2024-03-13-170000_sso_users_cascade`

**净差公式**：56 - 3 + 2 = **55** ✅

---

#### SQLite vs PostgreSQL：56 vs 46，净差 10 个

**SQLite 有而 PostgreSQL 没有的（14个）**：
```
2018-01-14-171611_create_tables
2018-02-17-205753_create_collections_and_orgs
2018-04-27-155151_create_users_ciphers
2018-05-08-161616_create_collection_cipher_map
2018-05-25-232323_update_attachments_reference
2018-06-01-112529_update_devices_twofactor_remember
2018-07-11-181453_create_u2f_twofactor
2018-08-27-172114_update_ciphers
2018-09-10-111213_add_invites
2018-09-19-144557_add_kdf_columns
2018-11-27-152651_add_att_key_columns
2019-05-26-216651_rename_key_and_type_columns
2024-02-14-140000_change_time_stamp_data_type
2024-03-13_170000_sso_userscascade
```

**PostgreSQL 有而 SQLite 没有的（4个）**：
1. `2019-09-12-100000_create_tables` — 起步时一次性创建完整表结构，替代上述前 11 个演进式迁移
2. `2019-09-16-150000_fix_attachments` — PostgreSQL 独有：CHAR→VARCHAR 修正，避免字符填充问题
3. `2024-02-14-135953_change_time_stamp_data_type` — 同一功能时间戳不同
4. `2024-03-13-170000_sso_users_cascade` — 同一功能命名不同

**净差公式**：56 - 14 + 4 = **46** ✅

---

#### MySQL vs PostgreSQL：55 vs 46，净差 9 个

**MySQL 有而 PostgreSQL 没有的（13个）**：
```
2018-01-14-171611_create_tables
2018-02-17-205753_create_collections_and_orgs
2018-04-27-155151_create_users_ciphers
2018-05-08-161616_create_collection_cipher_map
2018-05-25-232323_update_attachments_reference
2018-06-01-112529_update_devices_twofactor_remember
2018-07-11-181453_create_u2f_twofactor
2018-08-27-172114_update_ciphers
2018-09-10-111213_add_invites
2018-09-19-144557_add_kdf_columns
2018-11-27-152651_add_att_key_columns
2019-05-26-216651_rename_key_and_type_columns
2024-02-14-135828_change_time_stamp_data_type
```

**PostgreSQL 有而 MySQL 没有的（4个）**：
1. `2019-09-12-100000_create_tables`
2. `2019-09-16-150000_fix_attachments`
3. `2021-03-15-163412_rename_send_key` — PostgreSQL 与 SQLite 一致，先建 `key` 再改名
4. `2024-02-14-135953_change_time_stamp_data_type`

**净差公式**：55 - 13 + 4 = **46** ✅

---

#### "同名不同实"的迁移（命名差异但功能相同）

同一迁移在三种数据库中目录名称存在细微差异，是独立目录而非同一迁移：

| 功能 | SQLite | MySQL | PostgreSQL |
|------|--------|-------|------------|
| change_time_stamp_data_type | `140000` | `135828` | `135953` |
| sso_users_cascade | `2024-03-13_170000_sso_userscascade`<br/>(下划线分隔日期，无多余下划线) | `2024-03-13-170000_sso_users_cascade`<br/>(横线分隔日期，多一个下划线) | 同 MySQL |

### 2.2 代码层面的迁移执行

在 `src/db/mod.rs` 中，通过 `#[cfg]` 条件编译为每个数据库实现独立的迁移模块：

```rust
#[cfg(sqlite)]
mod sqlite_migrations {
    use diesel::{Connection, RunQueryDsl};
    use diesel_migrations::{EmbeddedMigrations, MigrationHarness};
    pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/sqlite");
    
    pub fn run_migrations(db_url: &str) -> Result<(), super::Error> {
        let mut connection = diesel::sqlite::SqliteConnection::establish(db_url)?;
        // SQLite 特有：禁用外键检查 + 启用 WAL
        diesel::sql_query("PRAGMA foreign_keys = OFF")
            .execute(&mut connection)
            .expect("Failed to disable Foreign Key Checks during migrations");
        if crate::CONFIG.enable_db_wal() {
            diesel::sql_query("PRAGMA journal_mode=wal")
                .execute(&mut connection)
                .expect("Failed to turn on WAL");
        }
        connection.run_pending_migrations(MIGRATIONS).expect("Error running migrations");
        Ok(())
    }
}

#[cfg(mysql)]
mod mysql_migrations {
    use diesel::{Connection, RunQueryDsl};
    use diesel_migrations::{EmbeddedMigrations, MigrationHarness};
    pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/mysql");
    
    pub fn run_migrations(db_url: &str) -> Result<(), super::Error> {
        let mut connection = diesel::mysql::MysqlConnection::establish(db_url)?;
        // MySQL 特有：禁用外键检查
        diesel::sql_query("SET FOREIGN_KEY_CHECKS = 0")
            .execute(&mut connection)
            .expect("Failed to disable Foreign Key Checks during migrations");
        connection.run_pending_migrations(MIGRATIONS).expect("Error running migrations");
        Ok(())
    }
}

#[cfg(postgresql)]
mod postgresql_migrations {
    use diesel::Connection;
    use diesel_migrations::{EmbeddedMigrations, MigrationHarness};
    pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/postgresql");
    
    pub fn run_migrations(db_url: &str) -> Result<(), super::Error> {
        let mut connection = diesel::pg::PgConnection::establish(db_url)?;
        // PostgreSQL 不禁用外键检查，依赖正确的迁移顺序
        connection.run_pending_migrations(MIGRATIONS).expect("Error running migrations");
        Ok(())
    }
}
```

### 2.3 迁移触发时机

迁移在 `DbPool::from_config()` 中自动执行：

1. 解析 `DATABASE_URL` 判断数据库类型
2. 根据数据库类型调用对应 `run_migrations()`
3. 创建连接池
4. 设置全局 `ACTIVE_DB_TYPE`

---

## 三、兼容边界分析

### 3.1 数据类型映射差异（已核准）

| 逻辑类型 | SQLite | MySQL | PostgreSQL |
|---------|--------|-------|------------|
| UUID | `TEXT` | `CHAR(36)` | `CHAR(36)` → 后改为 `VARCHAR(40)` |
| 字符串 | `TEXT` | `TEXT` / `VARCHAR(255)` | `TEXT` / `VARCHAR(255)` / `VARCHAR(40)` |
| 时间戳 | `DATETIME` | `DATETIME` | `TIMESTAMP` |
| 二进制 | `BLOB` | `BLOB` | `BYTEA` |
| 布尔值 | `BOOLEAN` | `BOOLEAN` | `BOOLEAN` |
| 大整数 | `INTEGER` (动态 i64) | `BIGINT` | `BIGINT` |

**PostgreSQL 特殊修正**：`2019-09-16-150000_fix_attachments` 迁移将所有 `CHAR(n)` 改为 `VARCHAR(n)` 或 `TEXT`，避免字符填充问题。

### 3.2 关键字处理策略（已核准）

#### 3.2.1 第一次大规模改名：2019-05-26

迁移 `2019-05-26-216651_rename_key_and_type_columns` 统一处理 SQL 关键字冲突：

**SQLite**:
```sql
ALTER TABLE attachments RENAME COLUMN key TO akey;
ALTER TABLE ciphers RENAME COLUMN type TO atype;
ALTER TABLE devices RENAME COLUMN type TO atype;
ALTER TABLE twofactor RENAME COLUMN type TO atype;
ALTER TABLE users RENAME COLUMN key TO akey;
ALTER TABLE users_organizations RENAME COLUMN key TO akey;
ALTER TABLE users_organizations RENAME COLUMN type TO atype;
```

**MySQL**（使用 `CHANGE COLUMN` 语法，关键字用反引号转义）:
```sql
ALTER TABLE attachments CHANGE COLUMN `key` akey TEXT;
ALTER TABLE ciphers CHANGE COLUMN type atype INTEGER NOT NULL;
ALTER TABLE devices CHANGE COLUMN type atype INTEGER NOT NULL;
ALTER TABLE twofactor CHANGE COLUMN type atype INTEGER NOT NULL;
ALTER TABLE users CHANGE COLUMN `key` akey TEXT;
ALTER TABLE users_organizations CHANGE COLUMN `key` akey TEXT;
ALTER TABLE users_organizations CHANGE COLUMN type atype INTEGER NOT NULL;
```

**PostgreSQL**：无此迁移，因为起步时（2019-09-12）就直接使用 `akey` 和 `atype` 命名。

#### 3.2.2 sends 表的特殊处理：2021-03

- **SQLite**: `2021-03-11` 创建 sends 表使用 `key` 列名，`2021-03-15` 通过 `RENAME COLUMN key TO akey` 改名
- **PostgreSQL**: 同上，两步完成
- **MySQL**: `2021-03-11` 创建 sends 表时直接使用 `akey` 列名，一步到位，无需后续改名迁移

#### 3.2.3 cipher.key 的特殊保留：2023-10-21

`2023-10-21-221242_add_cipher_key` 迁移为 ciphers 表添加 `key` 列，这次**刻意保留关键字命名**，通过数据库特定语法转义：

- **SQLite**: `ALTER TABLE ciphers ADD COLUMN "key" TEXT;` （双引号转义）
- **MySQL**: `ALTER TABLE ciphers ADD COLUMN `key` TEXT;` （反引号转义）
- **PostgreSQL**: `ALTER TABLE ciphers ADD COLUMN "key" TEXT;` （双引号转义）

**统一层映射**：在 `src/db/schema.rs` 中 Diesel 自动处理映射：
```rust
table! {
    ciphers (uuid) {
        ...
        key -> Nullable<Text>,  // Rust 代码中直接使用 key
        atype -> Integer,       // 其他列使用改名后的 atype
        ...
    }
}
```

### 3.3 ALTER TABLE 语法差异（已核准）

以 `2024-02-14` 时间戳迁移（修改 `twofactor.last_used` 列类型）为例：

**SQLite** (`2024-02-14-140000_change_time_stamp_data_type`):
```sql
-- SQLite 的 INTEGER 本身就是 i64 动态类型，无需修改
```

**MySQL** (`2024-02-14-135828_change_time_stamp_data_type`):
```sql
ALTER TABLE twofactor MODIFY last_used BIGINT NOT NULL;
```

**PostgreSQL** (`2024-02-14-135953_change_time_stamp_data_type`):
```sql
ALTER TABLE twofactor
ALTER COLUMN last_used TYPE BIGINT,
ALTER COLUMN last_used SET NOT NULL;
```

### 3.4 外键检查策略（已核准）

迁移期间的外键检查处理：

| 数据库 | 迁移期间处理 | 作用范围 |
|--------|-------------|----------|
| SQLite | `PRAGMA foreign_keys = OFF` | 连接级别 |
| MySQL | `SET FOREIGN_KEY_CHECKS = 0` | 会话级别 |
| PostgreSQL | 不禁用 | 依赖正确的迁移顺序 |

**PostgreSQL 的特殊性**：PostgreSQL 不支持在事务内禁用外键约束，因此必须保证迁移顺序的正确性。

### 3.5 连接初始化语句（已核准）

在 `DbConnType::default_init_stmts()` 中：

```rust
pub fn default_init_stmts(&self) -> String {
    match self {
        #[cfg(mysql)]
        Self::Mysql => String::new(),
        #[cfg(postgresql)]
        Self::Postgresql => String::new(),
        #[cfg(sqlite)]
        Self::Sqlite => "PRAGMA busy_timeout = 5000; PRAGMA synchronous = NORMAL;".to_owned(),
    }
}
```

**SQLite 独有配置**：
- `busy_timeout = 5000`：等待锁的超时时间 5 秒
- `synchronous = NORMAL`：平衡性能和安全性的同步级别
- 可选 `journal_mode = WAL`：Write-Ahead Logging 模式，提升并发性能

---

## 四、编译时特性系统

### 4.1 Cargo 特性定义

在 `Cargo.toml` 中：

```toml
[features]
default = []  # 默认不启用任何数据库

mysql = ["diesel/mysql", "diesel_migrations/mysql"]
postgresql = ["diesel/postgres", "diesel_migrations/postgres"]
sqlite_system = ["diesel/sqlite", "diesel_migrations/sqlite"]  # 动态链接
sqlite = ["sqlite_system", "libsqlite3-sys/bundled"]          # 静态链接
```

### 4.2 构建脚本配置

在 `build.rs` 中：

```rust
// 将 feature 转换为 cfg 标记，简化代码中的条件编译
#[cfg(feature = "sqlite_system")]
println!("cargo:rustc-cfg=sqlite");
#[cfg(feature = "mysql")]
println!("cargo:rustc-cfg=mysql");
#[cfg(feature = "postgresql")]
println!("cargo:rustc-cfg=postgresql");

// 至少启用一个数据库
#[cfg(not(any(feature = "sqlite_system", feature = "mysql", feature = "postgresql")))]
compile_error!("You need to enable one DB backend. To build with previous defaults do: cargo build --features sqlite");
```

---

## 五、运行时多态机制

### 5.1 MultiConnection 枚举

在 `src/db/mod.rs` 中定义：

```rust
#[derive(diesel::MultiConnection)]
pub enum DbConnInner {
    #[cfg(mysql)]
    Mysql(diesel::mysql::MysqlConnection),
    #[cfg(postgresql)]
    Postgresql(diesel::pg::PgConnection),
    #[cfg(sqlite)]
    Sqlite(diesel::sqlite::SqliteConnection),
}
```

### 5.2 db_run! 宏

`db_run!` 宏提供数据库特定代码的执行方式：

```rust
// 通用执行
db_run! { conn: {
    // 跨数据库通用代码
}}

// 按数据库类型分支执行
db_run! { conn:
    postgresql,mysql {
        // PostgreSQL 和 MySQL 特定代码
        diesel::select(diesel::dsl::sql::<Text>("version();"))
            .get_result::<String>(conn)
    }
    sqlite {
        // SQLite 特定代码
        diesel::select(diesel::dsl::sql::<Text>("sqlite_version();"))
            .get_result::<String>(conn)
    }
}
```

---

## 六、迁移链路流程图

```
启动
  │
  ▼
解析 DATABASE_URL
  │
  ├─► mysql:     ──► 检查 mysql feature ──► mysql_migrations::run_migrations()
  │                                                  │
  │                                                  ├─ SET FOREIGN_KEY_CHECKS = 0
  │                                                  └─ run_pending_migrations
  ├─► postgres:  ──► 检查 postgresql feature ──► postgresql_migrations::run_migrations()
  │                                                  │
  │                                                  └─ run_pending_migrations (不禁用外键)
  └─► sqlite:    ──► 检查 sqlite feature ──► sqlite_migrations::run_migrations()
                                                     │
                                                     ├─ PRAGMA foreign_keys = OFF
                                                     ├─ 可选 PRAGMA journal_mode=wal
                                                     └─ run_pending_migrations
                                                          │
                                                          ▼
                                          diesel_migrations::embed_migrations!()
                                                          │
                                                          ▼
                                          执行 __diesel_schema_migrations 表检查
                                                          │
                                                          ▼
                                          按时间戳顺序执行未执行的迁移
                                                          │
                                                          ▼
                                          创建连接池 + 设置 ACTIVE_DB_TYPE
                                                          │
                                                          ▼
                                                        就绪
```

---

## 七、关键兼容边界总结（已核准）

| 维度 | SQLite | MySQL | PostgreSQL |
|------|--------|-------|------------|
| **迁移起点** | 2018-01 | 2018-01 | 2019-09 (跳过早期演进) |
| **迁移数量** | 56 | 55 | 46 |
| **UUID 类型** | TEXT | CHAR(36) | CHAR(36) → VARCHAR(40) |
| **时间类型** | DATETIME | DATETIME | TIMESTAMP |
| **二进制类型** | BLOB | BLOB | BYTEA |
| **关键字 `key` 改名** | 2019-05 迁移 | 2019-05 迁移 | 起始即用 `akey` |
| **sends.key 改名** | 2021-03 迁移 `2021-03-15_rename_send_key` | 未发生，创建 sends 时直接用 `akey` | 2021-03 迁移 `2021-03-15_rename_send_key` |
| **cipher.key 新增** | 2023-10 `"key"` 转义 | 2023-10 `` `key` `` 转义 | 2023-10 `"key"` 转义 |
| **迁移时外键** | 禁用 | 禁用 | 保持启用 |
| **ALTER TABLE** | 常为空操作 | `MODIFY` | `ALTER COLUMN ... TYPE` |
| **连接初始化** | busy_timeout + synchronous | 无 | 无 |
| **WAL 模式** | 支持配置 | 不适用 | 不适用 |
| **数据库备份** | VACUUM INTO 支持 | 不支持 | 不支持 |

---

## 八、关键字改名时间线（已核准）

```
2018-01  SQLite/MySQL 创建表，使用 key/type 列名
2018-11  添加 attachments.key 列
   │
   ▼
2019-05  迁移 2019-05-26-216651_rename_key_and_type_columns
   │     ├─ SQLite: RENAME COLUMN key → akey, type → atype
   │     └─ MySQL: CHANGE COLUMN `key` → akey, type → atype
   │
2019-09  PostgreSQL 起步，2个独有迁移
         ├─ 2019-09-12_create_tables: 直接使用 akey/atype，跳过改名
         └─ 2019-09-16_fix_attachments: CHAR→VARCHAR 修正
   │
2021-03  创建 sends 表
   │     ├─ SQLite/PG: 使用 key → 4 天后通过 2021-03-15_rename_send_key 改名
   │     └─ MySQL: 直接使用 akey，无需改名（少1个迁移）
   │
2023-10  添加 ciphers.key 列，刻意保留关键字名
         ├─ SQLite: ADD COLUMN "key" TEXT
         ├─ MySQL: ADD COLUMN `key` TEXT
         └─ PG: ADD COLUMN "key" TEXT
         └─ schema.rs 统一映射为 key -> Nullable<Text>

2024-02  change_time_stamp_data_type: 三库目录名不同
         ├─ SQLite: 2024-02-14-140000_
         ├─ MySQL:  2024-02-14-135828_
         └─ PG:     2024-02-14-135953_
```

---

## 九、注意事项

1. **跨数据库数据迁移不支持**：三种数据库的迁移脚本完全独立，没有提供从一种数据库迁移到另一种数据库的工具

2. **编译时单数据库**：每个编译产物只能包含一种数据库支持，需在编译时通过 `--features` 指定

3. **回滚支持**：每个迁移目录都有 `down.sql`，但生产环境通常不建议回滚

4. **WAL 模式**：仅 SQLite 支持 WAL（Write-Ahead Logging）模式配置

5. **迁移顺序依赖**：PostgreSQL 不禁用外键，迁移顺序必须正确，SQLite 和 MySQL 则通过禁用外键规避顺序问题

6. **关键字处理原则**：
   - 早期列名：统一改名为 `akey`/`atype`
   - 新增列名：优先避免使用关键字
   - 业务必需保留关键字：通过数据库特定转义语法处理，Diesel schema 层统一映射
