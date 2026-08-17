# summer-rs Misc Plugins

其他官方插件：mail、opentelemetry、grpc、nacos、xxl-job、apalis、opendal、sa-token、test。

---

## summer-mail — SMTP 邮件

基于 [Lettre](https://github.com/lettre/lettre)。

### 依赖

```toml
summer-mail = { version = "0.7" }
```

### 配置

```toml
[mail]
host = "smtp.gmail.com"     # SMTP 服务器地址
port = 465                  # 端口
secure = true               # 启用 TLS
auth = { user = "user@gmail.com", password = "passwd" }
test_connection = false     # 启动时是否测试连接
```

### 注册的组件

`Mailer` — 别名 `lettre::AsyncSmtpTransport<Tokio1Executor>`。

```rust
pub type Mailer = lettre::AsyncSmtpTransport<Tokio1Executor>;
```

### 使用示例

```rust
use summer_web::{get, extractor::Component, error::Result};
use summer_web::axum::{response::IntoResponse, Json};
use summer_mail::Mailer;
use lettre::{Message, message::header::ContentType};
use anyhow::Context;

#[get("/send-mail")]
async fn send_mail(Component(mailer): Component<Mailer>) -> Result<impl IntoResponse> {
    let email = Message::builder()
        .from("NoBody <nobody@domain.tld>".parse().unwrap())
        .reply_to("Yuin <yuin@domain.tld>".parse().unwrap())
        .to("hff1996723@163.com".parse().unwrap())
        .subject("Happy new year")
        .header(ContentType::TEXT_PLAIN)
        .body("Be happy!".to_string())
        .unwrap();
    let resp = mailer.send(email).await.context("send mail failed")?;
    Ok(Json(resp))
}
```

完整示例：[mail-example](https://github.com/summer-rs/summer-rs/tree/master/examples/integrations/mail-example)

---

## summer-grpc — gRPC 服务

基于 [Tonic](https://github.com/hyperium/tonic)。

### 依赖

```toml
summer-grpc = { version = "0.7" }
tonic = "0.13"
prost = "0.13"

[build-dependencies]
tonic-build = "0.13"
```

### 配置

```toml
[grpc]
binding = "0.0.0.0"     # 绑定地址，默认 0.0.0.0
port = 9090             # 端口，默认 9090
graceful = true         # 优雅关闭，默认 false
```

### 服务实现

**1. 定义 proto**：

```proto
syntax = "proto3";
package helloworld;

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}
```

**2. 编译 proto**（`build.rs`）：

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/helloworld.proto")?;
    Ok(())
}
```

**3. 实现服务**：

```rust
use summer::plugin::service::Service;
use summer::App;
use summer_grpc::GrpcPlugin;
use tonic::{Request, Response, Status};

pub mod hello_world {
    tonic::include_proto!("helloworld");
}
use hello_world::greeter_server::{Greeter, GreeterServer};
use hello_world::{HelloReply, HelloRequest};

#[derive(Clone, Service)]
#[service(grpc = "GreeterServer")]
struct MyGreeter;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        println!("Request from {:?}", request.remote_addr());
        let reply = HelloReply {
            message: format!("Hello {}!", request.into_inner().name),
        };
        Ok(Response::new(reply))
    }
}

#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(GrpcPlugin)
        .run()
        .await;
}
```

关键点：
- `#[derive(Service)]` + `#[service(grpc = "GreeterServer")]` 让 `GrpcPlugin` 自动注册服务
- `Service` 也可注入 `#[inject(component)]` 和 `#[inject(config)]` 字段

完整示例：[grpc-example](https://github.com/summer-rs/summer-rs/tree/master/examples/grpc/grpc-example)

---

## summer-opentelemetry — 可观测性

集成 [OpenTelemetry Rust SDK](https://github.com/open-telemetry/opentelemetry-rust)。

### 依赖

```toml
summer-opentelemetry = { version = "0.7" }
```

可选 features：
- `jaeger` — 使用 Jaeger 格式传播上下文
- `zipkin` — 使用 Zipkin B3 格式传播上下文
- `more-resource` — 添加主机名、操作系统、进程信息等资源属性

默认使用 W3C Trace Context 格式传播。

### 配置

```toml
[opentelemetry]
enable = false    # 是否在运行时启用
```

其他配置建议使用 OpenTelemetry 规范中的环境变量，详见：
- [SDK Configuration](https://opentelemetry.io/docs/languages/sdk-configuration/)
- [Environment Variable Specification](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)

> **注意**：opentelemetry-rust 尚未稳定，tracing 集成的部分功能还在开发中。插件会持续跟踪相关动态并及时更新。

完整示例：[opentelemetry-example](https://github.com/summer-rs/summer-rs/tree/master/examples/integrations/opentelemetry-example)

---

## summer-nacos — 配置中心与服务发现

连接 [Nacos](https://nacos.io) 或 [r-nacos](https://github.com/nacos-group/r-nacos)。

使用官方 Rust 客户端 [`nacos-sdk`](https://crates.io/crates/nacos-sdk)（gRPC 2.x），兼容 r-nacos。

### 依赖

```toml
summer-nacos = { version = "0.7" }
```

### 配置

```toml
[nacos]
server_addr = "127.0.0.1:8848"
app_name = "my-app"
enable_config = true        # 启用配置中心
enable_naming = true        # 启用服务发现

# 启动时拉取的配置（按顺序合并，后面的覆盖前面的）
[[nacos.bootstrap]]
data_id = "app.toml"
group = "DEFAULT_GROUP"

[[nacos.bootstrap]]
data_id = "feature-flags.toml"
group = "DEFAULT_GROUP"

# 服务注册
[nacos.registration]
service_name = "my-app"
# port 不指定时从 summer-web 配置获取
```

### 注册的组件

| 组件 | 条件 |
|------|------|
| `NacosConfigService` | `enable_config = true` 时注册 |
| `NacosNamingService` | `enable_naming = true` 或设置了 `registration` 时注册 |

配合 `summer-web` 或 `summer-grpc` 使用时，设置 `registration` 后会自动在 `ServerStartedEvent` 时注册服务，并在 shutdown 时注销。

完整示例：[nacos-example](https://github.com/summer-rs/summer-rs/tree/master/examples/integrations/nacos-example)

---

## summer-xxl-job — 分布式任务执行器

集成 [`xxljob-sdk-rs`](https://crates.io/crates/xxljob-sdk-rs)，使应用成为 xxl-job-admin 或 [ratch-job](https://github.com/ratch-job/ratch-job) 的**执行器**。

### 依赖

```toml
summer-xxl-job = { version = "0.7" }
```

可选 features：`rustls-tls`（reqwest rustls）、`native-tls`（reqwest native tls）。

### 配置

```toml
[xxl-job]
admin_addresses = "http://127.0.0.1:8080/xxl-job-admin"
app_name        = "summer-xxl-executor"
access_token    = "default_token"
log_path        = "logs/xxl-job"
# 以下可选
# ip                = "10.0.0.5"
# port              = 9999
# log_retention_days = 30
# ssl_danger_accept_invalid_certs = false
# [xxl-job.headers]
# X-Gateway-Token = "my-token"
```

### 基本用法

```rust
use summer::{async_trait, App};
use summer_xxl_job::{AsyncJobHandler, JobContext, XxlJobConfigurator, XxlJobPlugin};

pub struct DemoJobHandler;

#[async_trait]
impl AsyncJobHandler for DemoJobHandler {
    async fn process(&self, ctx: JobContext) -> anyhow::Result<JobContext> {
        tracing::info!(job_id = ctx.job_id, "running demo handler");
        Ok(ctx)
    }
}

#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(XxlJobPlugin)
        .add_xxl_async_handler("demoJobHandler", DemoJobHandler)
        .run()
        .await;
}
```

### 依赖注入模式

需要注入组件/配置的 handler，使用 `#[derive(Service)]`：

```rust
use summer::plugin::service::Service;

#[derive(Clone, Service)]
pub struct DemoServiceHandler {
    #[inject(component)]
    xxl_client: XxlClientHandle,
    #[inject(config)]
    cfg: DemoJobConfig,
}

#[async_trait]
impl AsyncJobHandler for DemoServiceHandler {
    async fn process(&self, ctx: JobContext) -> anyhow::Result<JobContext> {
        tracing::info!(greeting = %self.cfg.greeting, "async DI handler");
        Ok(ctx)
    }
}

// 注册时用 add_xxl_async_service 而非 add_xxl_async_handler
App::new()
    .add_plugin(XxlJobPlugin)
    .add_xxl_async_service::<DemoServiceHandler>("demoServiceJobHandler")
    .run()
    .await;
```

### 注册 API 对照

| API | Handler trait | 构造方式 | 适用场景 |
|-----|---------------|----------|----------|
| `add_xxl_async_handler(name, value)` | `AsyncJobHandler` | 立即构造 | 无状态 handler |
| `add_xxl_sync_handler(name, value)` | `SyncJobHandler` | 立即构造 | 无状态同步 handler |
| `add_xxl_async_service::<H>(name)` | `AsyncJobHandler` + `Service` | DI 完成后延迟构造 | 需要注入依赖的异步 handler |
| `add_xxl_sync_service::<H>(name)` | `SyncJobHandler` + `Service` | DI 完成后延迟构造 | 需要注入依赖的同步 handler |

### 与 summer-job 共存

`summer-xxl-job` 只处理**远程**调度（xxl-job-admin / ratch-job）。本地 cron / fixed-rate / fixed-delay 任务仍由 `summer-job` 提供。两个插件独立，可同时启用。

完整示例：[xxl-job-example](https://github.com/summer-rs/summer-rs/tree/master/examples/job/xxl-job-example)

---

## summer-apalis — 后台任务

集成 [Apalis](https://github.com/apalis-dev/apalis) 后台任务处理框架。支持多种存储系统，也支持 cron 和步骤任务。

### 依赖

```toml
summer-apalis = { version = "0.7" }
```

完整示例：[apalis-redis-example](https://github.com/summer-rs/summer-rs/tree/master/examples/job/apalis-redis-example)

---

## summer-opendal — 统一对象存储

集成 [Apache OpenDAL](https://github.com/apache/opendal)，提供统一的文件和对象存储访问接口。支持 S3、本地文件系统等多种后端。

```toml
summer-opendal = { version = "0.7" }
```

---

## summer-sa-token — 权限认证

集成 [sa-token-rust](https://github.com/click33/sa-token-rust)，提供认证与授权功能。

该插件位于独立的 [contrib-plugins](https://github.com/summer-rs/contrib-plugins) 仓库中，**不参与 summer-rs 的 Cargo workspace**。使用 crates.io 上的版本。

```toml
summer-sa-token = { version = "0.7" }
```

---

## summer-test — 测试工具

提供基于 [axum-test](https://docs.rs/axum-test) 的 Web E2E 测试能力，无需绑定 TCP 端口。

### 依赖（dev-dependencies）

```toml
[dev-dependencies]
summer-test = { version = "0.7" }
summer-web = { version = "0.7" }
tokio = { version = "1", features = ["full", "macros"] }
```

### MockWebPlugin — 替代 WebPlugin 的内存测试

```rust
use summer::app::App;
use summer::plugin::ComponentRegistry;
use summer_test::{MockServer, MockWebPlugin};
use summer_web::{Router, WebConfigurator, axum::routing};

async fn ping() -> &'static str { "pong" }

#[tokio::test]
async fn ping_returns_pong() -> summer::error::Result<()> {
    App::new()
        .add_plugin(MockWebPlugin)
        .add_router(Router::new().route("/ping", routing::get(ping)))
        .build()
        .await?
        .get_expect_component::<MockServer>()
        .get("/ping")
        .await
        .assert_status_ok()
        .assert_text("pong");
    Ok(())
}
```

### 工作原理

1. **相同路由路径** — 复用 `assemble_router` / `finalize_router`，handler、中间件、layers、OpenAPI 行为与运行时一致
2. **内存服务器** — 将路由包装在 `axum_test::TestServer` 而非绑定端口
3. **不阻塞** — `build()` 完成后直接返回，不进入 serve loop
4. **插件兼容** — `MockWebPlugin.name()` 返回 `"summer_web::WebPlugin"`，依赖生产 Web 插件的其他插件可正常解析
5. **合成启动事件** — 发布 `ServerStartedEvent`（`127.0.0.1:0`, HTTP），让 `summer-nacos` 等服务发现插件正常初始化

### 与其他插件配合测试

```rust
App::new()
    .use_config_str(r#"
[postgres]
connect = "postgres://postgres:postgres@127.0.0.1:5432/postgres"
"#)
    .add_plugin(PgPlugin)
    .add_plugin(MockWebPlugin)
    .add_router(router())
    .build()
    .await?;
```

> 容器测试需要 Docker daemon。配合 [testcontainers](https://crates.io/crates/testcontainers) 可启动数据库容器做集成测试。

### API 概览

| 项 | 说明 |
|-----|------|
| `MockWebPlugin` | 替代 `WebPlugin` 的测试插件 |
| `MockServer` | 作为 App 组件存储的 `TestServer` handle，`Deref<Target = TestServer>` |
| `axum_test`（re-export） | 便捷的请求构建和断言方法 |

