---
name: summer-rs
description: >-
  This skill should be used when working with the summer-rs Rust application framework (a SpringBoot-inspired framework with plugin architecture, TOML configuration, and compile-time DI). 
  Activate when the user asks to create a summer-rs project, add plugins (web/SQLx/Redis/gRPC/job/stream/etc.), define REST APIs with summer-web, configure the application,
  implement dependency injection via #[component] or #[derive(Service)], set up cron jobs or message streams, integrate databases/Redis/Nacos/xxl-job/OpenTelemetry,
  write tests with summer-test, or troubleshoot summer-rs errors.
---

# summer-rs Framework Guide

summer-rs is a Rust application framework inspired by Java SpringBoot. Core crate: `summer` (v0.7.x). Official plugins use the `summer-*` naming convention.

- Official docs: https://summer-rs.github.io/
- Repository: https://github.com/summer-rs/summer-rs

## When to Use This Skill

Use this skill when the user mentions any of:

- **Creating a new summer-rs project** or adding summer-rs to an existing Rust project
- **Adding plugins**: `summer-web`, `summer-sqlx`, `summer-redis`, `summer-job`, `summer-stream`, `summer-grpc`, `summer-nacos`, `summer-xxl-job`, `summer-opentelemetry`, `summer-mail`, `summer-postgres`, `summer-sea-orm`, `summer-apalis`, `summer-opendal`, `summer-sa-token`, `summer-test`
- **Defining web routes** with `#[get]` / `#[post]` macros, extractors, middlewares, OpenAPI
- **Database integration**: SQLx, SeaORM, PostgreSQL, Redis with `#[cache]`
- **Background jobs**: `#[cron]`, `#[fix_delay]`, `#[fix_rate]`
- **Message streaming**: `#[stream_listener]` with Kafka/Redis/file backends
- **gRPC services** with Tonic integration
- **Configuration**: TOML setup, environment profiles, `#[config_prefix]`
- **Dependency injection**: `#[component]` factories, `#[derive(Service)]`, `#[inject]`
- **Service discovery**: Nacos integration
- **Distributed tasks**: xxl-job executor
- **Observability**: OpenTelemetry tracing/metrics
- **Testing**: summer-test with `MockWebPlugin`
- **Troubleshooting**: Config not found, component not found, cyclic dependencies, duplicate plugin errors

## Quick Start Workflow

When the user asks to create a summer-rs project:

1. **Scaffold the project structure** with the conventional layout:
   ```
   project/
   ├── Cargo.toml
   ├── config/
   │   └── app.toml
   └── src/
       └── main.rs
   ```

2. **Add dependencies** to `Cargo.toml` — `summer` core plus needed plugin crates (see plugin list below for crate names)

3. **Create `config/app.toml`** with plugin-specific sections (refer to individual plugin docs in `references/`)

4. **Write `src/main.rs`** using `#[auto_config]` + `App::new().add_plugin(...).run().await`

5. **Add handlers/jobs/components** in separate modules as needed

When the user asks about a specific plugin, load the corresponding reference file from `references/` for detailed configuration and usage.

## Core Concepts

The framework is built around three abstractions:

| Abstraction | Trait/Bound | Role |
|-------------|-------------|------|
| **Plugin** | `summer::plugin::Plugin` | Implement `async fn build(&self, app: &mut AppBuilder)` |
| **Config** | `summer::config::Configurable` + `serde::Deserialize` | Load from TOML with `#[config_prefix = "..."]` |
| **Component** | `Clone + Send + Sync + 'static` | Registered in DI container, extractable in handlers/jobs |

**Startup lifecycle** (`AppBuilder::inner_run`):
1. Print banner → 2. Build plugins (topological sort by dependencies, includes `#[component]` auto-registered plugins) → 3. Compile-time DI (`service::auto_inject_service`) → 4. Dispatch (web/job/stream schedulers)

**Two launch modes**:
- `App::new().run().await` — For schedulable apps (web, job, stream); internal loop never returns
- `App::new().build().await` — Returns `Result<Arc<App>>`; for CLI tools, tests, or apps without scheduling

## Project Structure Convention

```
project/
├── Cargo.toml
├── config/
│   ├── app.toml          # Default configuration
│   └── app-dev.toml      # Environment config (merged when SUMMER_ENV=dev, higher priority)
└── src/
    └── main.rs
```

**Configuration files**: Default reads `./config/app.toml`. Use `SUMMER_ENV` env var to auto-merge `app-{env}.toml` from the same directory. Environment config takes priority. Supports `${ENV_VAR:default}` interpolation.

**Important**: When running examples, always `cd` into the example directory (containing `config/` and `Cargo.toml`) before `cargo run`.

## Application Entry Point: `#[auto_config]`

The entry macro registers auto-discovered handlers and jobs:

```rust
use summer::{auto_config, App};

#[auto_config(WebConfigurator)]  // Auto-discovers all #[get]/#[post]/... routes
#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(WebPlugin)
        .add_plugin(SqlxPlugin)
        .run()
        .await;
}
```

- `#[auto_config(WebConfigurator)]` — Auto-collects all `#[get]`/`#[post]`/etc. annotated routes; no manual `add_router()` needed
- `#[auto_config(JobConfigurator)]` — Auto-collects `#[cron]`/`#[fix_delay]`/`#[fix_rate]` annotated jobs
- `#[auto_config(StreamConfigurator)]` — Auto-collects `#[stream_listener]` annotated consumers

Multiple configurators can be combined if the app uses web + job + stream simultaneously.

## Component Registration: `#[component]`

Apply `#[component]` to **factory functions** to auto-generate a Plugin and register via `inventory`. No manual `add_plugin()` needed.

```rust
use summer::component;
use summer::config::{Configurable, Config};
use serde::Deserialize;

#[derive(Clone, Configurable, Deserialize)]
#[config_prefix = "database"]
struct DbConfig { host: String, port: u16 }

#[derive(Clone)]
struct DbConnection { url: String }

#[component]
fn create_db_connection(Config(config): Config<DbConfig>) -> DbConnection {
    DbConnection { url: format!("{}:{}", config.host, config.port) }
}
```

**Supported parameter types**:

| Parameter | Purpose | Dependency tracking |
|-----------|---------|---------------------|
| `Config<T>` | Inject config (T must be `Configurable + Deserialize`) | None |
| `Component<T>` | Inject another component (creates dependency edge) | Auto-generated `dependencies()` |
| Mixed | `fn f(Config(c): Config<C>, Component(db): Component<D>)` | All `Component` params auto-listed as dependencies |

**Supported return types**: `T`, `Result<T, E>`, `async fn -> T`, `async fn -> Result<T, E>`

**Custom names** (for multiple same-type components):
```rust
#[component(name = "PrimaryDatabase")]
fn create_primary_db(Config(config): Config<PrimaryDbConfig>) -> PrimaryDb { ... }
```

**Explicit dependency injection**:
```rust
#[component]
fn create_repository(
    #[inject("PrimaryDatabase")] Component(db): Component<PrimaryDb>,
) -> UserRepository { ... }
```

**Large components**: Wrap in `Arc` via NewType pattern to reduce clone cost:
```rust
#[derive(Clone)]
struct LargeComponent { data: Arc<Vec<u8>> }
```

For detailed component macro docs including NewType pattern, cyclic dependencies, migration from manual Plugin, and common errors, see `references/plugins.md`.

## Service DI: `#[derive(Service)]`

Compile-time dependency injection at the struct level (inspired by Google Dagger). Complements `#[component]` (runtime factory level).

```rust
use summer::plugin::service::Service;
use summer_sqlx::ConnectPool;

#[derive(Clone, Configurable, Deserialize)]
#[config_prefix = "user"]
struct UserConfig { username: String, project: String }

#[derive(Clone, Service)]
struct UserService {
    #[inject(component)]
    db: ConnectPool,           // Injected from DI container
    #[inject(config)]
    config: UserConfig,        // Injected from TOML config
}
```

**Optional dependencies**: `#[inject(component)] db: Option<ConnectPool>` — receives `None` if component not found.

**Nested injection**: `UserService → OtherService → DatabaseService` auto-resolves transitively.

**Circular dependencies**: Use `LazyComponent<T>` (wraps as `Arc<RwLock<...>>`, call `.get()` to access):
```rust
#[derive(Clone, Service)]
struct UserService {
    other_service: LazyComponent<OtherService>,  // No #[inject] needed
}
```

Service also supports gRPC mode: `#[service(grpc = "GreeterServer")]` (see `references/misc_plugins.md` → summer-grpc).

## Configuration System

### Defining Config

```rust
use summer::config::Configurable;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "my-plugin"]
struct MyConfig { a: u32, b: bool }
```

TOML:
```toml
[my-plugin]
a = 10
b = true
```

### Reading Config

- **In plugins**: `app.get_config::<MyConfig>()` returns `Result<MyConfig>`
- **In Web handlers**: `summer_web::extractor::Config<MyConfig>` (implements Axum `FromRequestParts`)
- **In Job/Stream handlers**: `summer::extractor::Config<MyConfig>` (from summer core crate)

### Config Sources

- **Default**: `./config/app.toml` + `./config/app-{env}.toml` (controlled by `SUMMER_ENV`)
- **Custom file**: `app.use_config_file("path/to/config.toml")`
- **Inline string**: `app.use_config_str(r#"..."#)` — embed config in binary
- **Merge append**: `app.merge_config_str(r#"..."#)` — merge at runtime

### Environment Variable Interpolation

```toml
[sea-orm]
uri = "${DATABASE_URL:postgres://postgres:xxxx@localhost/postgres}"
```

- `${VAR}` — read env var, kept as-is if absent
- `${VAR:default}` — provide default value

### Schema Autocompletion

Add to TOML first line for VSCode Even Better TOML hints:
```toml
#:schema https://summer-rs.github.io/config-schema.json
```

## Built-in Logging Plugin

`LogPlugin` is built into `summer` core, auto-loaded first. No manual addition needed.

```toml
[logger]
enable = true
level = "info"                    # trace | debug | info | warn | error
format = "compact"                # compact | pretty | json
time_style = "local"              # system | uptime | local | utc | none
override_filter = "info,axum=debug"

[logger.file]
enable = true
dir = "./logs"
rotation = "daily"                # minutely | hourly | daily | never
max_log_files = 365
non_blocking = true
```

Add custom tracing layers:
```rust
App::new()
    .add_layer(my_custom_layer)
    .add_plugin(WebPlugin)
    .run()
    .await;
```

## Application Events

Plugins subscribe to lifecycle events via `app.listen()`:

| Event | Timing |
|-------|--------|
| `ConfigEvent` | After config loaded, before plugin build |
| `PluginsBuiltEvent` | After all plugins built |
| `ServicesInjectedEvent` | After compile-time DI complete |
| `AppBuiltEvent` | App build complete |
| `ServerStartedEvent` | Server started (Nacos uses this for auto-registration) |
| `ShutdownEvent` | App shutting down |

```rust
impl Plugin for MyPlugin {
    async fn build(&self, app: &mut AppBuilder) {
        app.listen(|event: ConfigEvent, app: &mut AppBuilder| async move {
            Ok(())
        });
    }
}
```

## Custom Plugin Development

```rust
use summer::async_trait;
use summer::config::{Configurable, ConfigRegistry};
use summer::plugin::Plugin;
use summer::app::AppBuilder;
use serde::Deserialize;

struct MyPlugin;

#[async_trait]
impl Plugin for MyPlugin {
    async fn build(&self, app: &mut AppBuilder) {
        let config = app.get_config::<MyConfig>().expect("load config failed");
        // Build and register components based on config
    }
}

#[derive(Debug, Configurable, Deserialize)]
#[config_prefix = "my-plugin"]
struct MyConfig { a: u32, b: bool }
```

Component registration methods:
- `app.add_component(component)` — direct registration
- `#[component]` macro — declarative auto-registration

Key rules:
- Components must implement `Clone`; use NewType + Arc for large structs
- Plugins declare dependencies via `dependencies()`; framework builds in topological order
- Duplicate plugin names cause panic

## Complete Plugin List

| Plugin | Crate | Library | Registered Component(s) |
|--------|-------|---------|------------------------|
| Web | `summer-web` | Axum | Routes/Middlewares |
| SQLx | `summer-sqlx` | SQLx | `ConnectPool` (alias `sqlx::AnyPool`) |
| PostgreSQL | `summer-postgres` | tokio-postgres | `Postgres` (`Arc<tokio_postgres::Client>`) |
| SeaORM | `summer-sea-orm` | SeaORM | `DbConn` (alias `sea_orm::DbConn`) |
| Redis | `summer-redis` | redis-rs | `Redis` (alias `redis::aio::ConnectionManager`) |
| Mail | `summer-mail` | Lettre | `Mailer` (alias `AsyncSmtpTransport<Tokio1Executor>`) |
| Job | `summer-job` | tokio-cron-scheduler | Cron scheduler |
| Stream | `summer-stream` | sea-streamer | Producer / Consumer |
| gRPC | `summer-grpc` | Tonic | gRPC services |
| OpenTelemetry | `summer-opentelemetry` | OpenTelemetry | Traces/Metrics/Logs |
| Nacos | `summer-nacos` | nacos-sdk | `NacosConfigService` / `NacosNamingService` |
| xxl-job | `summer-xxl-job` | xxljob-sdk-rs | Distributed task executor |
| Apalis | `summer-apalis` | Apalis | Background jobs |
| OpenDAL | `summer-opendal` | OpenDAL | Unified object storage |
| Sa-Token | `summer-sa-token` | sa-token-rust | Auth/Authz |
| Test | `summer-test` | axum_test | `MockWebPlugin` / `MockServer` |

> For detailed configuration and usage of each plugin, load the corresponding file in `references/`.

## Reference Files

Load as needed for detailed information:

| File | Content |
|------|---------|
| `references/web_plugin.md` | summer-web: routes, extractors, middlewares, OpenAPI, ProblemDetails, Socket.IO |
| `references/data_plugins.md` | summer-sqlx, summer-postgres, summer-sea-orm (pagination), summer-redis (`#[cache]` macro) |
| `references/job_stream_plugins.md` | summer-job (cron/fix_delay/fix_rate), summer-stream (file/redis/kafka/stdio) |
| `references/misc_plugins.md` | summer-mail, summer-grpc, summer-opentelemetry, summer-nacos, summer-xxl-job, summer-apalis, summer-opendal, summer-sa-token, summer-test |
| `references/plugins.md` | Plugin system internals, Component macro internals, Config system internals, NewType patterns |
| `references/troubleshooting.md` | Common errors and solutions |
| `references/recipes.md` | Common implementation patterns (CRUD API, auth flow, file upload, pagination, etc.) |

## Common Patterns

For step-by-step patterns on building common features:
- **CRUD API with database** → `references/data_plugins.md` + `references/web_plugin.md`
- **Authentication flow** → `references/recipes.md`
- **File upload** → `references/web_plugin.md` (multipart feature)
- **Pagination** → `references/data_plugins.md` (SeaORM pagination section)
- **Scheduled background jobs** → `references/job_stream_plugins.md`
- **Distributed task execution** → `references/misc_plugins.md` (xxl-job section)
- **Service discovery / config center** → `references/misc_plugins.md` (Nacos section)
- **Testing** → `references/misc_plugins.md` (summer-test section)

## Troubleshooting Quick Reference

| Error | Likely Cause | Solution |
|-------|-------------|---------|
| `Config X not found` | Missing TOML section | Add `[prefix]` section to `config/app.toml` |
| `Component X not found` | Component not registered | Ensure dependency uses `#[component]` or `app.add_component()` |
| `Cyclic dependency detected` | Circular component graph | Refactor design; use `LazyComponent<T>` to break cycle |
| `plugin was already added` | Duplicate plugin name | Use NewType or custom name for same-type components |
| `component was already added` | Same type registered twice | Each type can only be registered once; use NewType pattern |

For detailed error resolution, see `references/troubleshooting.md`.

