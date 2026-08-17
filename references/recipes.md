# summer-rs Common Patterns & Recipes

Step-by-step patterns for implementing common features with summer-rs.

---

## Pattern 1: Full CRUD REST API

### Setup

```toml
# Cargo.toml
[dependencies]
summer = "0.7"
summer-web = "0.7"
summer-sqlx = { version = "0.7", features = ["postgres"] }
serde = { version = "1", features = ["derive"] }
anyhow = "1"
```

```toml
# config/app.toml
[web]
port = 8080
[web.middlewares]
compression = { enable = true }
logger = { enable = true, level = "info" }

[sqlx]
uri = "${DATABASE_URL:postgres://postgres:postgres@localhost:5432/myapp}"
max_connections = 10
```

### Model + Handler

```rust
// src/main.rs
use summer::{auto_config, App};
use summer_web::{WebPlugin, WebConfigurator, get, post, put, delete};
use summer_web::extractor::{Component, Path, Config};
use summer_web::axum::{Json, response::IntoResponse, http::StatusCode};
use summer_sqlx::{SqlxPlugin, ConnectPool, sqlx::{self, Row}};
use serde::{Deserialize, Serialize};
use anyhow::Context;

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
struct Item {
    id: i32,
    name: String,
    description: String,
}

#[derive(Debug, Deserialize)]
struct CreateItem { name: String, description: String }

#[get("/items")]
async fn list_items(Component(pool): Component<ConnectPool>) -> Result<Json<Vec<Item>>> {
    let items = sqlx::query_as::<_, Item>("SELECT id, name, description FROM items")
        .fetch_all(&pool).await.context("query failed")?;
    Ok(Json(items))
}

#[get("/items/{id}")]
async fn get_item(
    Component(pool): Component<ConnectPool>,
    Path(id): Path<i32>,
) -> Result<Json<Item>> {
    let item = sqlx::query_as::<_, Item>("SELECT id, name, description FROM items WHERE id = $1")
        .bind(id).fetch_one(&pool).await.context("not found")?;
    Ok(Json(item))
}

#[post("/items")]
async fn create_item(
    Component(pool): Component<ConnectPool>,
    Json(body): Json<CreateItem>,
) -> Result<(StatusCode, Json<Item>)> {
    let item = sqlx::query_as::<_, Item>(
        "INSERT INTO items (name, description) VALUES ($1, $2) RETURNING id, name, description"
    ).bind(&body.name).bind(&body.description).fetch_one(&pool).await.context("insert failed")?;
    Ok((StatusCode::CREATED, Json(item)))
}

#[put("/items/{id}")]
async fn update_item(
    Component(pool): Component<ConnectPool>,
    Path(id): Path<i32>,
    Json(body): Json<CreateItem>,
) -> Result<Json<Item>> {
    let item = sqlx::query_as::<_, Item>(
        "UPDATE items SET name = $1, description = $2 WHERE id = $3 RETURNING id, name, description"
    ).bind(&body.name).bind(&body.description).bind(id).fetch_one(&pool).await.context("update failed")?;
    Ok(Json(item))
}

#[delete("/items/{id}")]
async fn delete_item(
    Component(pool): Component<ConnectPool>,
    Path(id): Path<i32>,
) -> Result<StatusCode> {
    sqlx::query("DELETE FROM items WHERE id = $1")
        .bind(id).execute(&pool).await.context("delete failed")?;
    Ok(StatusCode::NO_CONTENT)
}

type Result<T> = std::result::Result<T, summer_web::error::Error>;

#[auto_config(WebConfigurator)]
#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(WebPlugin)
        .add_plugin(SqlxPlugin)
        .run()
        .await;
}
```

### Using SeaORM instead of raw SQLx

Replace the handler implementations with SeaORM entity queries. See `references/data_plugins.md` for the SeaORM entity pattern.

---

## Pattern 2: Service Layer with Dependency Injection

### Structure

```
src/
├── main.rs
├── config.rs       # Config structs
├── models.rs       # Data models / entities
├── services/
│   ├── mod.rs
│   ├── user_service.rs
│   └── order_service.rs
└── handlers/
    ├── mod.rs
    ├── user_handler.rs
    └── order_handler.rs
```

### Service Implementation

```rust
// src/services/user_service.rs
use summer::plugin::service::Service;
use summer::config::Configurable;
use summer_sqlx::ConnectPool;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "user"]
pub struct UserConfig { pub default_role: String }

#[derive(Clone, Service)]
pub struct UserService {
    #[inject(component)]
    db: ConnectPool,
    #[inject(config)]
    config: UserConfig,
}

impl UserService {
    pub async fn find_by_id(&self, id: i32) -> anyhow::Result<User> {
        // ... query using self.db and self.config ...
        todo!()
    }
}
```

```rust
// src/handlers/user_handler.rs
use summer_web::{get, extractor::Component};
use summer_web::axum::Json;
use crate::services::user_service::UserService;

#[get("/users/{id}")]
async fn get_user(Component(svc): Component<UserService>) -> Json<User> {
    let user = svc.find_by_id(1).await.unwrap();
    Json(user)
}
```

---

## Pattern 3: Authentication Middleware

### Module-Level Middleware

```rust
use summer_web::{
    middlewares, get,
    axum::middleware,
    axum::response::{Response, IntoResponse},
    axum::middleware::Next,
    axum::http::StatusCode,
    extractor::{Request, Component},
};
use summer_sqlx::ConnectPool;
use summer::config::Configurable;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "auth"]
struct AuthConfig { jwt_secret: String }

#[middlewares(middleware::from_fn(auth_middleware))]
mod secured_routes {
    use super::*;

    async fn auth_middleware(
        Component(db): Component<ConnectPool>,
        mut request: Request,
        next: Next,
    ) -> Response {
        let auth_header = request.headers()
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "));

        match auth_header {
            Some(token) if !token.is_empty() => {
                // Validate token, extract user ID, insert into request extensions
                request.extensions_mut().insert(UserId(42));
                next.run(request).await
            }
            _ => (StatusCode::UNAUTHORIZED, "Missing or invalid token").into_response(),
        }
    }

    #[derive(Clone)]
    struct UserId(i32);

    #[get("/me")]
    async fn whoami(
        Component(db): Component<ConnectPool>,
        // Access extensions set by middleware
        request: Request,
    ) -> impl IntoResponse {
        let user_id = request.extensions().get::<UserId>().unwrap();
        format!("User ID: {}", user_id.0)
    }
}
```

### Config-Driven Middleware (Global)

```toml
[web.middlewares]
compression = { enable = true }
catch_panic = { enable = true }
logger = { enable = true, level = "info" }
limit_payload = { enable = true, body_limit = "5MB" }
```

For custom global middleware, add tower Layers programmatically in the plugin build method.

---

## Pattern 4: Scheduled Background Job

```rust
use summer::{auto_config, App, extractor::Component};
use summer_job::{JobPlugin, JobConfigurator, cron};
use summer_sqlx::{SqlxPlugin, ConnectPool};

#[cron("0 */5 * * * *")]  // Every 5 minutes
async fn cleanup_expired_sessions(
    Component(pool): Component<ConnectPool>,
) {
    let result = sqlx::query("DELETE FROM sessions WHERE expires_at < NOW()")
        .execute(&pool)
        .await;
    match result {
        Ok(r) => tracing::info!(deleted = r.rows_affected(), "cleaned expired sessions"),
        Err(e) => tracing::error!(error = %e, "failed to clean sessions"),
    }
}

#[auto_config(JobConfigurator)]
#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(JobPlugin)
        .add_plugin(SqlxPlugin)
        .run()
        .await;
}
```

---

## Pattern 5: Redis Caching

```rust
use summer_redis::cache;

#[cache("user:{user_id}", expire = 300, condition = user_id > 0)]
async fn get_user_profile(user_id: i64) -> anyhow::Result<UserProfile> {
    // Expensive database query, cached for 5 minutes
    // Only cache if user_id > 0
    let profile = query_user_from_db(user_id).await?;
    Ok(profile)
}
```

Requirements:
- Function must be `async`
- Return type must implement `Serialize + Deserialize`
- Redis plugin must be added: `app.add_plugin(RedisPlugin)`

---

## Pattern 6: File Upload

### Enable multipart feature

```toml
summer-web = { version = "0.7", features = ["multipart"] }
```

### Handler

```rust
use summer_web::{post, axum::extract::Multipart};
use summer_web::axum::response::IntoResponse;

#[post("/upload")]
async fn upload_file(mut multipart: Multipart) -> impl IntoResponse {
    while let Some(field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let file_name = field.file_name().unwrap().to_string();
        let data = field.bytes().await.unwrap();
        // Save data to disk or object storage...
        tracing::info!(name, file_name, size = data.len(), "file uploaded");
    }
    "upload complete"
}
```

---

## Pattern 7: Paginated API with SeaORM

See `references/data_plugins.md` for full SeaORM pagination configuration and usage with `PaginationExt`.

---

## Pattern 8: gRPC Service with Dependencies

See `references/misc_plugins.md` (summer-grpc section) for the complete pattern using `#[service(grpc = "...")]` with `#[derive(Service)]`.

---

## Pattern 9: Nacos Service Discovery + Web

```toml
# config/app.toml
[web]
port = 8080

[nacos]
server_addr = "127.0.0.1:8848"
enable_naming = true

[nacos.registration]
service_name = "my-app"
# port auto-detected from web config
```

```rust
// main.rs
App::new()
    .add_plugin(NacosPlugin)
    .add_plugin(WebPlugin)
    .run()
    .await;
```

The app auto-registers with Nacos on `ServerStartedEvent` and deregisters on shutdown.

---

## Pattern 10: Distributed Task with xxl-job + DI

```rust
use summer::plugin::service::Service;
use summer_xxl_job::{AsyncJobHandler, JobContext, XxlJobPlugin};
use summer_sqlx::ConnectPool;

#[derive(Clone, Service)]
pub struct OrderCleanupHandler {
    #[inject(component)]
    db: ConnectPool,
}

#[async_trait::async_trait]
impl AsyncJobHandler for OrderCleanupHandler {
    async fn process(&self, ctx: JobContext) -> anyhow::Result<JobContext> {
        // Use self.db to query and clean expired orders
        tracing::info!(job_id = ctx.job_id, "processing order cleanup");
        Ok(ctx)
    }
}

// In main:
App::new()
    .add_plugin(XxlJobPlugin)
    .add_plugin(SqlxPlugin)
    .add_xxl_async_service::<OrderCleanupHandler>("orderCleanupJob")
    .run()
    .await;
```

---

## Pattern 11: Integration Testing

```rust
#[cfg(test)]
mod tests {
    use summer::App;
    use summer_test::{MockWebPlugin, MockServer};
    use summer_web::{Router, get, axum::routing};

    #[get("/health")]
    async fn health() -> &'static str { "ok" }

    #[tokio::test]
    async fn health_check_returns_ok() {
        let app = App::new()
            .add_plugin(MockWebPlugin)
            .add_router(Router::new().route("/health", routing::get(health)))
            .build()
            .await
            .unwrap();

        app.get_expect_component::<MockServer>()
            .get("/health")
            .await
            .assert_status_ok()
            .assert_text("ok");
    }

    #[tokio::test]
    async fn test_with_real_db_via_config_str() {
        let app = App::new()
            .use_config_str(r#"
                [sqlx]
                uri = "postgres://postgres:postgres@127.0.0.1:5432/test_db"
                max_connections = 2
            "#)
            .add_plugin(SqlxPlugin)
            .add_plugin(MockWebPlugin)
            .add_router(my_router())
            .build()
            .await
            .unwrap();

        app.get_expect_component::<MockServer>()
            .get("/items")
            .await
            .assert_status_ok();
    }
}
```

---

## Pattern 12: Custom Plugin with Config + Component

```rust
use summer::async_trait;
use summer::config::{Configurable, ConfigRegistry};
use summer::plugin::{Plugin, ComponentRegistry};
use summer::app::AppBuilder;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "sms"]
pub struct SmsConfig { api_key: String, endpoint: String }

#[derive(Clone)]
pub struct SmsClient { api_key: String, endpoint: String }

impl SmsClient {
    pub async fn send(&self, phone: &str, msg: &str) -> anyhow::Result<()> {
        // ... HTTP call to SMS provider ...
        Ok(())
    }
}

pub struct SmsPlugin;

#[async_trait]
impl Plugin for SmsPlugin {
    async fn build(&self, app: &mut AppBuilder) {
        let config = app.get_config::<SmsConfig>()
            .expect("sms config not found in app.toml");
        let client = SmsClient {
            api_key: config.api_key,
            endpoint: config.endpoint,
        };
        app.add_component(client);
    }

    fn name(&self) -> &str { "sms_plugin" }
}
```

Usage in handler:
```rust
#[get("/send-sms")]
async fn send_sms(Component(sms): Component<SmsClient>) -> impl IntoResponse {
    sms.send("13800138000", "Hello").await.unwrap();
    "sent"
}
```

