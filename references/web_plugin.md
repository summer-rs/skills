# summer-web — Web 框架

基于 [Axum](https://github.com/tokio-rs/axum)，是 summer-rs 最核心的插件。Axum 基于 hyper + tokio + tower 生态构建。

## 依赖

```toml
summer-web = { version = "0.7" }
```

**可选 features**：
- `http2` — HTTP/2 支持
- `multipart` — 文件上传
- `ws` — WebSocket
- `socket_io` — Socket.IO 支持（集成 [socketioxide](https://github.com/Totodore/socketioxide)）
- `openapi` — OpenAPI 文档生成
- `openapi-redoc` / `openapi-scalar` / `openapi-swagger` — 文档 UI（三选一）

## 配置项

```toml
[web]
binding = "0.0.0.0"          # 绑定地址，默认 0.0.0.0
port = 8080                   # 端口，默认 8080
connect_info = false          # 客户端连接信息，默认 false
graceful = true               # 优雅关闭，默认 false
global_prefix = "api"         # 全局路由前缀，默认空

# 中间件配置
[web.middlewares]
compression = { enable = true }                         # 压缩
catch_panic = { enable = true }                         # 捕获 handler panic
logger = { enable = true, level = "info" }              # 请求日志
limit_payload = { enable = true, body_limit = "5MB" }   # 请求体大小限制
timeout_request = { enable = true, timeout = 60000 }    # 请求超时（毫秒）

# 跨域配置
cors = { enable = true, allow_origins = [
    "https://summer-rs.github.io",
], allow_headers = [
    "Authentication",
], allow_methods = [
    "GET", "POST",
], max_age = 60 }

# 静态资源
static = { enable = true, uri = "/static", path = "static",
           precompressed = true, fallback = "index.html" }
```

> 中间件基于 [tower](https://docs.rs/tower) 和 [tower-http](https://docs.rs/tower-http) 生态，也可通过代码自定义。

## 属性宏 — 路由定义

summer-web 提供所有标准 HTTP 方法的属性宏：

| 宏 | 等价 HTTP 方法 |
|----|---------------|
| `#[get("/path")]` | GET |
| `#[post("/path")]` | POST |
| `#[patch("/path")]` | PATCH |
| `#[put("/path")]` | PUT |
| `#[delete("/path")]` | DELETE |
| `#[head("/path")]` | HEAD |
| `#[trace("/path")]` | TRACE |
| `#[options("/path")]` | OPTIONS |

**多方法绑定**：
```rust
use summer_web::route;

#[route("/test", method = "GET", method = "HEAD")]
async fn example() -> impl IntoResponse {
    "hello world"
}
```

**多路由绑定到同一 handler**：
```rust
use summer_web::{routes, get, delete};

#[routes]
#[get("/test")]
#[get("/test2")]
#[delete("/test")]
async fn example() -> impl IntoResponse {
    "hello world"
}
```

## 手动路由注册

不使用 `#[auto_config]` 时，需手动构造 Router：

```rust
use summer::App;
use summer_web::{
    WebPlugin, WebConfigurator, Router,
    handler::TypeRouter,
};

#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(WebPlugin)
        .add_router(router())
        .run()
        .await;
}

fn router() -> Router {
    Router::new()
        .typed_route(hello_world)
}

#[get("/")]
async fn hello_world() -> impl IntoResponse {
    "hello world"
}
```

使用 `#[auto_config(WebConfigurator)]` 后无需手动 `add_router()`——路由自动发现。

## 提取器（Extractor）

### Component — 从 DI 容器提取组件

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

### Config — 从 TOML 配置提取

```rust
use summer::config::Configurable;
use summer_web::{get, extractor::Config};
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "custom"]
struct CustomConfig {
    a: u32,
    b: bool,
}

#[get("/config")]
async fn use_config(Config(conf): Config<CustomConfig>) -> impl IntoResponse {
    format!("a={}, b={}", conf.a, conf.b)
}
```

TOML 配置：
```toml
[custom]
a = 1
b = true
```

### Axum 内置提取器

`summer_web::extractor` 还 re-export 了 Axum 的所有标准提取器：
- `Path<T>` — URL 路径参数
- `Query<T>` — URL 查询参数
- `Json<T>` — JSON 请求体
- `Form<T>` — 表单数据
- `State<T>` — 应用状态
- `Request` — 原始 HTTP 请求
- `Extension<T>` — 请求扩展

## 中间件

### 声明式中间件

使用 `#[middlewares]` 宏在模块级别应用中间件：

```rust
use summer_web::{middlewares, get, axum::middleware};
use summer_web::axum::{response::Response, middleware::Next, response::IntoResponse};
use summer_web::extractor::{Request, Component};
use summer_sqlx::ConnectPool;

#[middlewares(
    middleware::from_fn(auth_middleware),
)]
mod routes {
    use super::*;

    async fn auth_middleware(
        Component(db): Component<ConnectPool>,
        request: Request,
        next: Next,
    ) -> Response {
        // 验证逻辑...
        let response = next.run(request).await;
        response
    }

    #[get("/secured")]
    async fn secured() -> impl IntoResponse {
        "authenticated"
    }
}
```

> 注意：中间件函数中的提取器必须遵循 Axum 的规则（即提取器需实现 `FromRequestParts`）。

### 全局中间件

通过 `[web.middlewares]` 配置启用（见上文配置项），也可代码中添加 tower Layer。

## OpenAPI 文档

### 启用

1. 启用 `openapi` feature + 一个 UI feature（`openapi-redoc` / `openapi-scalar` / `openapi-swagger`）
2. 使用 `_api` 后缀的宏定义 API 端点

### API 宏

对应标准 HTTP 方法：`get_api`、`post_api`、`patch_api`、`put_api`、`delete_api`、`head_api`、`trace_api`、`options_api`。

通用宏：`api_route`（类似 `route`，同时绑定多个方法）。

```rust
use summer_web::{get_api, axum::Json};

/// # 获取用户列表
/// 返回系统中所有用户的信息。
///
/// 支持分页查询。
/// @tag 用户管理
/// @status_codes Errors::NotFound, Errors::Internal
#[get_api("/users")]
async fn list_users() -> Json<Vec<User>> {
    // ...
    Json(vec![])
}
```

**文档注释约定**：
- `# 标题`（Markdown h1）— API 的 summary
- 其他注释内容 — API 描述
- `@tag tag名` — 分组标签
- `@status_codes Errors::X` — 声明可能返回的错误类型

### ProblemDetails 错误处理

`#[derive(ProblemDetails)]` 自动为自定义错误类型生成 `From<T> for ProblemDetails` 和 `IntoResponse` 实现：

```rust
use summer_web::ProblemDetails;
use summer_web::axum::http::StatusCode;

#[derive(thiserror::Error, Debug, ProblemDetails)]
pub enum ApiErrors {
    #[status_code(400)]
    #[error("Validation failed")]
    ValidationError,

    #[status_code(401)]
    #[problem_type("https://api.myapp.com/problems/auth-required")]
    #[error("Authentication required")]
    AuthenticationError,

    #[status_code(404)]
    #[problem_type("https://api.myapp.com/problems/not-found")]
    #[title("Resource Not Found")]
    #[detail("The requested resource could not be found")]
    #[instance("/users/{id}")]
    #[error("Resource not found")]
    NotFoundError,

    #[status_code(500)]
    #[error(transparent)]
    SqlxError(#[from] summer_sqlx::sqlx::Error),

    #[status_code(418)]
    #[error("TeaPod error: {0:?}")]
    TeaPod(CustomErrorSchema),
}
```

**支持的属性**：
- `#[status_code(code)]` — **必需**，HTTP 状态码
- `#[problem_type("uri")]` — 可选，自定义 problem type URI
- `#[title("title")]` — 可选，自定义标题（默认从 `#[error]` 推导）
- `#[detail("detail")]` — 可选，详细描述
- `#[instance("uri")]` — 可选，实例 URI（框架自动捕获当前请求 URI）

**常用状态码自动映射**：
- `400` → `ProblemDetails::validation_error()`
- `401` → `ProblemDetails::authentication_error()`
- `403` → `ProblemDetails::authorization_error()`
- `404` → `ProblemDetails::not_found()`
- `500` → `ProblemDetails::internal_server_error()`
- `503` → `ProblemDetails::service_unavailable()`

生成的 JSON 响应格式（RFC 7807）：
```json
{
  "type": "https://api.myapp.com/problems/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "The requested resource could not be found",
  "instance": "/users/999"
}
```

## Socket.IO 支持

启用 `socket_io` feature，集成 [socketioxide](https://github.com/Totodore/socketioxide)。

支持命名事件、自动重连、心跳、房间/命名空间、传输降级。

Handler 中可以像普通 HTTP handler 一样使用 `Component` 提取器注入组件。

参考示例：[socket-io-example](https://github.com/summer-rs/summer-rs/tree/master/examples/web/socket-io-example)

## 与 Axum 的关系

summer-web 是 Axum 的薄包装，添加了过程宏简化开发。所有 [Axum 示例](https://github.com/tokio-rs/axum/tree/main/examples) 都可在 summer-web 中运行。

如需直接使用 Axum 原语，从 `summer_web::axum::*` 导入。

