# summer-rs 插件体系参考

summer-rs 插件系统是其核心架构支柱。所有插件实现 `summer::plugin::Plugin` trait，配置实现 `summer::config::Configurable` trait，组件实现 `Clone` trait。

## Plugin Trait

```rust
use summer::async_trait;
use summer::plugin::Plugin;
use summer::app::AppBuilder;

#[async_trait]
pub trait Plugin: Send + Sync {
    /// 插件构建方法，在应用启动时被调用
    async fn build(&self, app: &mut AppBuilder) {
        // default: no-op
    }

    /// 插件名称，用于去重和依赖解析
    fn name(&self) -> &str {
        std::any::type_name::<Self>()
    }

    /// 依赖的其他插件名称列表
    fn dependencies(&self) -> Vec<&str> {
        vec![]
    }

    /// 是否立即构建（跳过依赖排序，在注册时立即执行 build）
    fn immediately(&self) -> bool {
        false
    }

    /// 立即构建的快捷方法
    fn immediately_build(&self, app: &mut AppBuilder) {
        // ...
    }
}
```

## 插件构建顺序

框架按如下步骤构建插件：

1. `add_auto_plugins()` — 收集所有通过 `#[component]` 宏注册的插件（`inventory::iter::<&dyn Plugin>`）
2. 发布 `ConfigEvent`
3. 构建 `LogPlugin`（内置，最先执行）
4. 按**拓扑排序**构建其余插件（依赖的插件先构建，循环依赖会 panic）
5. 发布 `PluginsBuiltEvent`

## 配置系统

### Configurable Trait

```rust
use summer::config::Configurable;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "my-plugin"]
struct MyConfig {
    a: u32,
    b: bool,
}
```

- `#[config_prefix]` 指定 TOML 中的 section 名称
- 必须同时实现 `Deserialize`
- 必须实现 `Clone`（因为配置会被多个组件持有）

### 配置加载

配置从 TOML 文件加载，支持环境变量插值：

```toml
[my-plugin]
a = 10
b = true

[sea-orm]
uri = "${DATABASE_URL:postgres://localhost/test}"
```

环境变量格式：
- `${VAR}` — 读取环境变量，不存在保留原样
- `${VAR:default_value}` — 提供默认值

### 读取配置的方式

**插件内部**（`Plugin::build` 方法中）：
```rust
let config = app.get_config::<MyConfig>().expect("load config failed");
```

**请求处理器中**（Web）：
```rust
use summer_web::extractor::Config;

#[get("/config")]
async fn handler(Config(conf): Config<MyConfig>) -> impl IntoResponse {
    // ...
}
```

**定时任务中**（Job）：
```rust
use summer::extractor::Config;

#[cron("0/10 * * * * *")]
async fn job(Config(conf): Config<MyConfig>) {
    // ...
}
```

**流处理器中**（Stream）：
```rust
use summer::extractor::Config;

#[stream_listener("topic")]
async fn consumer(Config(conf): Config<MyConfig>) {
    // ...
}
```

### 环境管理

通过 `SUMMER_ENV` 环境变量控制当前环境：

```rust
use summer::app::Env;

let env = app.get_env();  // Env::Dev, Env::Test, Env::Prod, Env::Custom(...)
```

框架会自动加载 `./config/app-{env}.toml` 覆盖默认配置。环境枚举：
- `Env::Dev` — SUMMER_ENV=dev
- `Env::Test` — SUMMER_ENV=test  
- `Env::Prod` — SUMMER_ENV=prod
- `Env::Custom(str)` — 自定义环境名

## 组件系统

### 组件注册

组件是注入到 DI 容器中的对象，必须实现 `Clone + Send + Sync + 'static`。

**方式一：插件中手动注册**

```rust
impl Plugin for MyPlugin {
    async fn build(&self, app: &mut AppBuilder) {
        app.add_component(MyService::new());
    }
}
```

同一类型只能注册一次，重复注册会 panic。

**方式二：`#[component]` 宏自动注册**

参见 SKILL.md 中 Component 宏章节。

### 组件获取

```rust
// 获取组件（返回 Option）
let service = app.get_component::<MyService>();

// 获取组件（不存在时 panic）
let service = app.get_expect_component::<MyService>();

// 检查组件是否存在
if app.has_component::<MyService>() { ... }
```

### NewType 模式（推荐）

由于组件基于 `TypeId` 注册，同一类型只能注册一次。需要多个同类型实例时使用 NewType 模式：

```rust
#[derive(Clone)]
struct PrimaryDb(DbConnection);

#[derive(Clone)]
struct SecondaryDb(DbConnection);

#[component(name = "PrimaryDatabase")]
fn create_primary_db(Config(config): Config<PrimaryDbConfig>) -> PrimaryDb {
    PrimaryDb(DbConnection::new(&config))
}

#[component(name = "SecondaryDatabase")]
fn create_secondary_db(Config(config): Config<SecondaryDbConfig>) -> SecondaryDb {
    SecondaryDb(DbConnection::new(&config))
}
```

大型组件也用 NewType 包装 `Arc<T>` 减少 clone 开销：

```rust
#[derive(Clone)]
struct BigService(Arc<InnerBigService>);

impl BigService {
    fn new() -> Self {
        BigService(Arc::new(InnerBigService::new()))
    }
}
```

## Component 宏详解

`#[component]` 是简化组件注册的过程宏，应用于**返回组件实例的函数**。

### 工作原理

1. 生成一个匿名 Plugin 实现
2. 在 `build` 方法中从 `AppBuilder` 提取配置和依赖组件
3. 调用原函数创建组件实例
4. 通过 `app.add_component()` 注册
5. 通过 `inventory::submit!` 自动向框架注册该插件

### 完整示例

```rust
use summer::component;
use summer::config::{Configurable, Config};
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "database"]
struct DbConfig {
    host: String,
    port: u16,
    max_connections: u32,
}

#[derive(Clone)]
struct Pool {
    config: DbConfig,
}

// 同步初始化
#[component]
fn create_pool(Config(config): Config<DbConfig>) -> Pool {
    Pool { config }
}

// 异步 + 可失败初始化
#[component]
async fn create_pool_async(
    Config(config): Config<DbConfig>,
) -> Result<Pool, anyhow::Error> {
    // 异步连接验证等
    Ok(Pool { config })
}

// 依赖多个组件和配置
#[derive(Clone)]
struct AppService {
    pool: Pool,
    redis: Redis,
    config: AppConfig,
}

#[component]
fn create_app_service(
    Config(config): Config<AppConfig>,
    Component(pool): Component<Pool>,
    Component(redis): Component<Redis>,
) -> AppService {
    AppService { pool, redis, config }
}
```

### 最佳实践

1. **工厂函数保持简洁** — 只做创建和配置，不含业务逻辑
2. **配置驱动** — 所有可变值通过 Config 注入，不要硬编码
3. **使用 Result 返回** — 可失败初始化用 `Result<T, E>`，避免内部 unwrap
4. **文档化依赖关系** — 在 factory 函数上用 doc comment 说明依赖

### 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `Config X not found` | 配置缺失 | 在 `config/app.toml` 添加对应配置 |
| `Component X not found` | 组件未注册 | 确保依赖组件也用 `#[component]` 标注 |
| `Cyclic dependency detected` | 循环依赖 | 重构设计去掉循环 |
| `plugin was already added` | 同名插件/同类型组件重复 | 使用 NewType 或自定义名称 |
| `component was already added` | 同类型组件多次注册 | 每种类型只能注册一次，用 NewType |

## 配置驱动的自定义插件开发

```rust
use summer::async_trait;
use summer::config::{Configurable, ConfigRegistry};
use summer::plugin::{Plugin, ComponentRegistry};
use summer::app::AppBuilder;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "my-service"]
struct MyServiceConfig {
    endpoint: String,
    timeout_ms: u64,
}

#[derive(Clone)]
struct MyService {
    config: MyServiceConfig,
}

struct MyServicePlugin;

#[async_trait]
impl Plugin for MyServicePlugin {
    async fn build(&self, app: &mut AppBuilder) {
        let config = app.get_config::<MyServiceConfig>()
            .expect("MyServiceConfig not found in config");
        let service = MyService { config };
        app.add_component(service);
    }

    fn name(&self) -> &str {
        "my_service_plugin"
    }
}
```

```toml
# config/app.toml
[my-service]
endpoint = "https://api.example.com"
timeout_ms = 5000
```

