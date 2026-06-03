# 多数据库迁移链路分析

## 一、整体架构概览

Vaultwarden 使用 Diesel ORM 框架实现多数据库支持，通过**编译时特性标记**和**独立迁移目录**的方式，实现 SQLite、PostgreSQL、MySQL 三种数据库的兼容。

### 1.1 核心组件位置

| 组件 | 文件位置 |
|------|----------|
| 数据库连接管理 | [src/db/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/mod.rs) |
| SQLite 迁移目录 | [migrations/sqlite/](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/sqlite) |
| MySQL 迁移目录 | [migrations/mysql/](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/mysql) |
| PostgreSQL 迁移目录 | [migrations/postgresql/](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/postgresql) |
| 编译特性配置 | [Cargo.toml](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/Cargo.toml) |
| 构建脚本 | [build.rs](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/build.rs) |

---

## 二、迁移组织方式

### 2.1 目录结构

```
migrations/
├── sqlite/          # 57 个迁移脚本 (2018-01 ~ 2026-05)
│   ├── 2018-01-14-171611_create_tables/
│   ├── ...
│   └── 2026-05-05-120000_sso_auth_error/
├── mysql/           # 56 个迁移脚本 (2018-01 ~ 2026-05)
│   ├── 2018-01-14-171611_create_tables/
│   ├── ...
│   └── 2026-05-05-120000_sso_auth_error/
└── postgresql/      # 47 个迁移脚本 (2019-09 ~ 2026-05)
    ├── 2019-09-12-100000_create_tables/
    ├── ...
    └── 2026-05-05-120000_sso_auth_error/
```

**关键差异**：
- **PostgreSQL 起步较晚**：第一个迁移从 2019-09 开始，直接创建完整表结构，而非像 SQLite/MySQL 那样逐步演进
- **迁移命名不完全对齐**：同一功能的迁移在不同数据库中可能有细微的时间戳差异
- **部分迁移只存在于特定数据库**：如 `2021-03-15-163412_rename_send_key` 在 SQLite 和 PostgreSQL 中有，但 MySQL 中合并到了其他迁移

### 2.2 代码层面的迁移执行

在 [src/db/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/mod.rs#L474-L536) 中，通过 `#[cfg]` 条件编译为每个数据库实现独立的迁移模块：

```rust
#[cfg(sqlite)]
mod sqlite_migrations {
    pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/sqlite");
    
    pub fn run_migrations(db_url: &str) -> Result<(), Error> {
        let mut connection = diesel::sqlite::SqliteConnection::establish(db_url)?;
        // SQLite 特有：禁用外键检查 + 启用 WAL
        diesel::sql_query("PRAGMA foreign_keys = OFF").execute(&mut connection)?;
        if CONFIG.enable_db_wal() {
            diesel::sql_query("PRAGMA journal_mode=wal").execute(&mut connection)?;
        }
        connection.run_pending_migrations(MIGRATIONS).expect("...");
        Ok(())
    }
}

#[cfg(mysql)]
mod mysql_migrations {
    pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/mysql");
    
    pub fn run_migrations(db_url: &str) -> Result<(), Error> {
        let mut connection = diesel::mysql::MysqlConnection::establish(db_url)?;
        // MySQL 特有：禁用外键检查
        diesel::sql_query("SET FOREIGN_KEY_CHECKS = 0").execute(&mut connection)?;
        connection.run_pending_migrations(MIGRATIONS).expect("...");
        Ok(())
    }
}

#[cfg(postgresql)]
mod postgresql_migrations {
    pub const MIGRATIONS: EmbeddedMigrations = embed_migrations!("migrations/postgresql");
    
    pub fn run_migrations(db_url: &str) -> Result<(), Error> {
        let mut connection = diesel::pg::PgConnection::establish(db_url)?;
        // PostgreSQL 不禁用外键检查
        connection.run_pending_migrations(MIGRATIONS).expect("...");
        Ok(())
    }
}
```

### 2.3 迁移触发时机

迁移在 [DbPool::from_config()](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/mod.rs#L182-L232) 中自动执行：

1. 解析 `DATABASE_URL` 判断数据库类型
2. 根据数据库类型调用对应 `run_migrations()`
3. 创建连接池
4. 设置全局 `ACTIVE_DB_TYPE`

---

## 三、兼容边界分析

### 3.1 数据类型映射差异

| 逻辑类型 | SQLite | MySQL | PostgreSQL |
|---------|--------|-------|------------|
| UUID | `TEXT` | `CHAR(36)` | `CHAR(36)` |
| 字符串 | `TEXT` | `TEXT` / `VARCHAR(255)` | `TEXT` / `VARCHAR(255)` |
| 时间戳 | `DATETIME` | `DATETIME` | `TIMESTAMP` |
| 二进制 | `BLOB` | `BLOB` | `BYTEA` |
| 布尔值 | `BOOLEAN` | `BOOLEAN` | `BOOLEAN` |
| 大整数 | `INTEGER` (自动 i64) | `BIGINT` | `BIGINT` |

**示例对比**（users 表）：

**SQLite** [migrations/sqlite/2018-01-14-171611_create_tables/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/sqlite/2018-01-14-171611_create_tables/up.sql#L1-L19):
```sql
CREATE TABLE users (
  uuid                TEXT     NOT NULL PRIMARY KEY,
  created_at          DATETIME NOT NULL,
  password_hash       BLOB     NOT NULL,
  salt                BLOB     NOT NULL,
  key                 TEXT     NOT NULL,  -- 关键字无需处理
  ...
);
```

**MySQL** [migrations/mysql/2018-01-14-171611_create_tables/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/mysql/2018-01-14-171611_create_tables/up.sql#L1-L19):
```sql
CREATE TABLE users (
  uuid                CHAR(36) NOT NULL PRIMARY KEY,
  created_at          DATETIME NOT NULL,
  password_hash       BLOB     NOT NULL,
  salt                BLOB     NOT NULL,
  `key`               TEXT     NOT NULL,  -- 使用反引号转义关键字
  ...
);
```

**PostgreSQL** [migrations/postgresql/2019-09-12-100000_create_tables/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/postgresql/2019-09-12-100000_create_tables/up.sql#L1-L21):
```sql
CREATE TABLE users (
  uuid                CHAR(36) NOT NULL PRIMARY KEY,
  created_at          TIMESTAMP NOT NULL,
  password_hash       BYTEA     NOT NULL,
  salt                BYTEA     NOT NULL,
  akey                TEXT     NOT NULL,  -- 重命名避免关键字冲突
  ...
);
```

### 3.2 关键字处理策略

三种数据库对 SQL 关键字 `key` 的处理方式不同：

| 数据库 | 处理方式 | 列名 |
|--------|---------|------|
| SQLite | 无需特殊处理 | `key` |
| MySQL | 反引号转义 | `` `key` `` |
| PostgreSQL | 重命名为 `akey` | `akey` |

**统一层**：在 [src/db/schema.rs](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/schema.rs) 和模型中使用统一的 Diesel 属性映射：

```rust
// schema.rs 中统一使用 akey
table! {
    ciphers (uuid) {
        ...
        key -> Nullable<Text>,  // schema 中使用 key
        ...
    }
}

// 模型通过 Diesel 属性映射
#[derive(Identifiable, Queryable, Insertable, AsChangeset)]
#[diesel(table_name = users)]
pub struct User {
    pub akey: String,  // Rust 代码中使用 akey
    ...
}
```

### 3.3 ALTER TABLE 语法差异

以 `2024-02-14` 时间戳迁移（修改 `last_used` 列类型）为例：

**SQLite** [migrations/sqlite/2024-02-14-140000_change_time_stamp_data_type/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/sqlite/2024-02-14-140000_change_time_stamp_data_type/up.sql#L1-L1):
```sql
-- SQLite 的 INTEGER 本身就是 i64，无需操作
```

**MySQL** [migrations/mysql/2024-02-14-135828_change_time_stamp_data_type/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/mysql/2024-02-14-135828_change_time_stamp_data_type/up.sql#L1-L1):
```sql
ALTER TABLE twofactor MODIFY last_used BIGINT NOT NULL;
```

**PostgreSQL** [migrations/postgresql/2024-02-14-135953_change_time_stamp_data_type/up.sql](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/migrations/postgresql/2024-02-14-135953_change_time_stamp_data_type/up.sql#L1-L3):
```sql
ALTER TABLE twofactor
ALTER COLUMN last_used TYPE BIGINT,
ALTER COLUMN last_used SET NOT NULL;
```

### 3.4 外键检查策略

迁移期间的外键检查处理：

| 数据库 | 迁移期间处理 |
|--------|-------------|
| SQLite | `PRAGMA foreign_keys = OFF` |
| MySQL | `SET FOREIGN_KEY_CHECKS = 0` |
| PostgreSQL | 不禁用，依赖正确的迁移顺序 |

### 3.5 连接初始化语句

在 [DbConnType::default_init_stmts()](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/mod.rs#L310-L319) 中：

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

---

## 四、编译时特性系统

### 4.1 Cargo 特性定义

在 [Cargo.toml](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/Cargo.toml#L24-L37) 中：

```toml
[features]
default = []  # 默认不启用任何数据库

mysql = ["diesel/mysql", "diesel_migrations/mysql"]
postgresql = ["diesel/postgres", "diesel_migrations/postgres"]
sqlite_system = ["diesel/sqlite", "diesel_migrations/sqlite"]
sqlite = ["sqlite_system", "libsqlite3-sys/bundled"]  # 静态链接
```

### 4.2 构建脚本配置

在 [build.rs](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/build.rs#L1-L14) 中：

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
compile_error!("You need to enable one DB backend...");
```

---

## 五、运行时多态机制

### 5.1 MultiConnection 枚举

在 [src/db/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/mod.rs#L45-L53) 中定义：

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

[db_run! 宏](file:///d:/fz/0601/solo-dogfeeding/code/14-vaultwarden/src/db/mod.rs#L337-L354) 提供数据库特定代码的执行方式：

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
  ├─► postgres:  ──► 检查 postgresql feature ──► postgresql_migrations::run_migrations()
  └─► sqlite:    ──► 检查 sqlite feature ──► sqlite_migrations::run_migrations()
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

## 七、关键兼容边界总结

| 维度 | SQLite | MySQL | PostgreSQL |
|------|--------|-------|------------|
| **迁移起点** | 2018-01 | 2018-01 | 2019-09 (跳过早期演进) |
| **迁移数量** | 57 | 56 | 47 |
| **UUID 类型** | TEXT | CHAR(36) | CHAR(36) |
| **时间类型** | DATETIME | DATETIME | TIMESTAMP |
| **二进制类型** | BLOB | BLOB | BYTEA |
| **关键字 `key`** | 直接使用 | 反引号转义 | 重命名为 `akey` |
| **迁移时外键** | 禁用 | 禁用 | 保持启用 |
| **ALTER TABLE** | 常为空操作 | MODIFY | ALTER COLUMN |
| **连接初始化** | busy_timeout + synchronous | 无 | 无 |
| **数据库备份** | VACUUM INTO 支持 | 不支持 | 不支持 |

---

## 八、注意事项

1. **跨数据库数据迁移不支持**：三种数据库的迁移脚本完全独立，没有提供从一种数据库迁移到另一种数据库的工具

2. **编译时单数据库**：每个编译产物只能包含一种数据库支持，需在编译时通过 `--features` 指定

3. **回滚支持**：每个迁移目录都有 `down.sql`，但生产环境通常不建议回滚

4. **WAL 模式**：仅 SQLite 支持 WAL（Write-Ahead Logging）模式配置
