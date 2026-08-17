# summer-rs Data Layer Plugins

数据库和缓存插件：summer-sqlx、summer-postgres、summer-sea-orm、summer-redis。

---

## summer-sqlx — 异步 SQL

集成 [SQLx](https://github.com/launchbadge/sqlx)，异步 SQL 库，编译期验证 SQL。

### 依赖

```toml
summer-sqlx = { version = "0.7", features = ["postgres"] }
```

数据库 driver features：
- `postgres` — PostgreSQL
- `mysql` — MySQL
- `sqlite` — SQLite

可选 features：`with-json`、`with-chrono`、`with-rust_decimal`、`with-bigdecimal`、`with-uuid`、`with-time`。

### 配置

```toml
[sqlx]
uri = "postgres://root:123456@localhost:5432/pg_db"
min_connections = 1          # 最小连接数，默认 1
max_connections = 10         # 最大连接数，默认 10
acquire_timeout = 30000      # 连接超时（毫秒），默认 30s
idle_timeout = 600000        # 连接空闲时间（毫秒），默认 10min
connect_timeout = 1800000    # 连接最大存活时间（毫秒），默认 30min
```

### 注册的组件

`ConnectPool` — 别名 `sqlx::AnyPool`。

```rust
pub type ConnectPool = sqlx::AnyPool;
```

### 使用示例

```rust
use summer_web::{get, extractor::Component, error::Result};
use summer_web::axum::response::IntoResponse;
use summer_sqlx::{ConnectPool, sqlx::{self, Row}};
use anyhow::Context;

#[get("/version")]
async fn db_version(Component(pool): Component<ConnectPool>) -> Result<String> {
    let version = sqlx::query("select version() as version")
        .fetch_one(&pool)
        .await
        .context("sqlx query failed")?
        .get("version");
    Ok(version)
}
```

完整示例：[hello-world-example](https://github.com/summer-rs/summer-rs/tree/master/examples/getting-started/hello-world-example)

---

## summer-postgres — PostgreSQL

集成 [tokio-postgres](https://github.com/sfackler/rust-postgres)，专注于 PostgreSQL 连接。

### 依赖

```toml
summer-postgres = { version = "0.7" }
```

可选 features：`with-chrono-0_4`、`with-serde_json-1`、`with-uuid-0_8`、`with-uuid-1`、`with-time-0_3`、`with-geo-types-0_7`、`with-eui48-0_4`、`with-eui48-1` 等。

### 配置

```toml
[postgres]
connect = "postgres://root:12341234@localhost:5432/myapp_development"
```

### 注册的组件

`Postgres` — NewType 包装 `Arc<tokio_postgres::Client>`。

```rust
pub struct Postgres(Arc<tokio_postgres::Client>);
```

### 使用示例

```rust
use summer_web::{get, extractor::Component, error::Result};
use summer_web::axum::{response::IntoResponse, Json};
use summer_postgres::Postgres;
use anyhow::Context;

#[get("/postgres")]
async fn version(Component(pg): Component<Postgres>) -> Result<impl IntoResponse> {
    let rows = pg
        .query("select version() as version", &[])
        .await
        .context("query postgresql failed")?;
    let version: String = rows[0].get("version");
    Ok(Json(version))
}
```

完整示例：[postgres-example](https://github.com/summer-rs/summer-rs/tree/master/examples/data/postgres-example)

---

## summer-sea-orm — ORM

集成 [SeaORM](https://www.sea-ql.org/SeaORM/)，基于 SQLx 的异步 ORM。

### 依赖

```toml
summer-sea-orm = { version = "0.7", features = ["postgres"] }
sea-orm = "1.0"  # 配合 sea-orm-cli 生成的 entity 代码
```

数据库 driver：`postgres`、`mysql`、`sqlite`。

可选 features：`with-web`（启用 web 分页参数解析）。

### 配置

```toml
[sea-orm]
uri = "postgres://root:123456@localhost:5432/pg_db"
min_connections = 1
max_connections = 10
acquire_timeout = 30000
idle_timeout = 600000
connect_timeout = 1800000
enable_logging = true      # 打印 SQL 日志
```

### 注册的组件

`DbConn` — 别名 `sea_orm::DbConn`。

```rust
pub type DbConn = sea_orm::DbConn;
```

### 使用示例

```rust
use summer_web::{get, extractor::{Component, Path}, error::Result};
use summer_web::axum::Json;
use summer_sea_orm::DbConn;
use anyhow::Context;

#[get("/{id}")]
async fn get_todos(
    Component(db): Component<DbConn>,
    Path(id): Path<i32>,
) -> Result<Json<Vec<TodoItem>>> {
    let rows = TodoItem::find()
        .filter(todo_item::Column::ListId.eq(id))
        .all(&db)
        .await
        .context("query failed")?;
    Ok(Json(rows))
}
```

### 分页支持

`summer-sea-orm` 扩展了 SeaORM 的 `Select`，提供 `PaginationExt` trait。

启用 `with-web` feature 后的 Web 分页配置：

```toml
[sea-orm-web]
one_indexed = false          # 是否基于 1 的索引，默认 false
max_page_size = 2000         # 最大页大小，防止 OOM，默认 2000
default_page_size = 20       # 默认页大小，默认 20
```

使用：

```rust
use summer_sea_orm::pagination::PaginationExt;

#[get("/")]
async fn paged_list(
    Component(db): Component<DbConn>,
    pagination: Pagination,
) -> Result<impl IntoResponse> {
    let rows = TodoList::find()
        .page(&db, pagination)
        .await
        .context("query failed")?;
    Ok(Json(rows))
}
```

### 实体代码生成

使用 [sea-orm-cli](https://www.sea-ql.org/SeaORM/docs/generate-entity/sea-orm-cli/) 从数据库表结构自动生成 Entity 代码。

完整示例：[sea-orm-example](https://github.com/summer-rs/summer-rs/tree/master/examples/data/sea-orm-example)

---

## summer-redis — Redis

集成 [redis-rs](https://github.com/redis-rs/redis-rs)。

### 依赖

```toml
summer-redis = { version = "0.7" }
```

### 配置

```toml
[redis]
uri = "redis://127.0.0.1/"
connection_timeout = 10000   # 连接超时（毫秒）
response_timeout = 1000      # 响应超时（毫秒）
number_of_retries = 6        # 重试次数（指数退避）
exponent_base = 2            # 退避指数基数（毫秒）
factor = 100                 # 增长因子
max_delay = 60000            # 最大退避间隔
```

### 注册的组件

`Redis` — 别名 `redis::aio::ConnectionManager`。

```rust
pub type Redis = redis::aio::ConnectionManager;
```

### 使用示例

```rust
use summer_web::{get, extractor::Component, error::Result};
use summer_web::axum::{response::IntoResponse, Json};
use summer_redis::Redis;
use anyhow::Context;

#[get("/redis-keys")]
async fn list_keys(Component(mut conn): Component<Redis>) -> Result<impl IntoResponse> {
    let keys: Vec<String> = conn.keys("*").await.context("redis request failed")?;
    Ok(Json(keys))
}
```

### `#[cache]` 宏

提供基于 Redis 的透明缓存，作用于 async 函数：

```rust
use summer_redis::cache;

#[cache("redis-cache:{key}", expire = 60, condition = key.len() > 3)]
async fn cachable_func(key: &str) -> String {
    format!("cached value for key: {key}")
}
```

**参数**：
- `"redis-key-pattern:{arg}"` — Redis key 模式（支持函数参数插值）
- `expire = 60` — 过期时间（秒）
- `condition = expr` — 条件表达式，为 true 时才缓存
- `unless = expr` — 条件表达式，为 true 时跳过缓存

**函数要求**：
- 必须是 `async fn`
- 返回 `Result<T, E>` 或普通值 `T`
- 返回类型必须实现 `serde::Serialize + serde::Deserialize`（底层使用 `serde_json`）

完整示例：[redis-example](https://github.com/summer-rs/summer-rs/tree/master/examples/data/redis-example)

