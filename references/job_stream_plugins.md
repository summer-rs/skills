# summer-rs Job & Stream 插件

---

## summer-job — 定时任务 / Cron

基于 [tokio-cron-scheduler](https://github.com/mvniekerk/tokio-cron-scheduler)。提供三种调度方式：`#[cron]`、`#[fix_delay]`、`#[fix_rate]`。

### 依赖

```toml
summer-job = { version = "0.7" }
```

### 基本用法

```rust
use summer::{auto_config, App};
use summer_job::{cron, fix_delay, fix_rate, JobConfigurator, JobPlugin, Jobs};
use summer_sqlx::SqlxPlugin;
use std::time::{Duration, SystemTime};

#[auto_config(JobConfigurator)]
#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(JobPlugin)
        .add_plugin(SqlxPlugin)
        .run()
        .await;

    tokio::time::sleep(Duration::from_secs(100)).await;
}
```

### 三种调度宏

**`#[cron("expr")]`** — Cron 表达式调度（6 字段：秒 分 时 日 月 周）：

```rust
use summer_job::cron;

#[cron("1/10 * * * * *")]  // 每 10 秒执行
async fn cron_job() {
    println!("cron scheduled: {:?}", SystemTime::now())
}
```

**`#[fix_delay(seconds)]`** — 固定延迟（上一次执行完成后等待 N 秒）：

```rust
use summer_job::fix_delay;

#[fix_delay(5)]  // 上次完成后等 5 秒再执行
async fn fix_delay_job() {
    println!("fix delay scheduled: {:?}", SystemTime::now())
}
```

**`#[fix_rate(seconds)]`** — 固定频率（每隔 N 秒执行，不等待上次完成）：

```rust
use summer_job::fix_rate;

#[fix_rate(5)]  // 每 5 秒执行
async fn fix_rate_job() {
    println!("fix rate scheduled: {:?}", SystemTime::now())
}
```

### 提取组件和配置

**注意**：`summer::extractor::Component` 已统一为 Job/Stream/DI 场景的 extractor 来源。但 Web handler 中仍使用 `summer_web::extractor::Component`（实现了 Axum 的 `FromRequestParts`），两者不可混用。

```rust
use anyhow::Context;
use summer_job::cron;
use summer::extractor::{Component, Config};
use summer_sqlx::{ConnectPool, sqlx::{self, Row}};
use summer::config::Configurable;
use serde::Deserialize;

#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "custom"]
struct CustomConfig {
    a: u32,
    b: bool,
}

#[cron("1/10 * * * * *")]
async fn cron_job(
    Component(db): Component<ConnectPool>,
    Config(conf): Config<CustomConfig>,
) {
    let time: String = sqlx::query("select DATE_FORMAT(now(),'%Y-%m-%d %H:%i:%s') as time")
        .fetch_one(&db)
        .await
        .context("query failed")
        .unwrap()
        .get("time");
    println!("cron scheduled: {:?}, a={}", time, conf.a)
}
```

TOML：
```toml
[custom]
a = 1
b = true
```

### 手动注册（不使用 #[auto_config]）

```rust
use summer_job::{Jobs, JobConfigurator};

fn jobs() -> Jobs {
    Jobs::new()
        .typed_job(cron_job)
        .typed_job(fix_delay_job)
}

#[tokio::main]
async fn main() {
    App::new()
        .add_plugin(JobPlugin)
        .add_plugin(SqlxPlugin)
        .add_jobs(jobs())
        .run()
        .await;
}
```

完整示例：[job-example](https://github.com/summer-rs/summer-rs/tree/master/examples/job/job-example)

---

## summer-stream — 消息流

基于 [sea-streamer](https://github.com/SeaQL/sea-streamer)。支持四种消息后端：`file`、`stdio`、`redis`、`kafka`。

### 依赖

```toml
summer-stream = { version = "0.7", features = ["file"] }
```

后端 features：`file`、`stdio`、`redis`、`kafka`。

可选 features：`json`（启用 JSON 消息序列化）。

### 配置

```toml
[stream]
uri = "file://./stream"    # StreamerUri 数据流地址
```

URI 格式参照 [StreamerUri](https://docs.rs/sea-streamer/latest/sea_streamer/struct.StreamerUri.html)：

- `file://./path` — 文件流（适合单机部署）
- `stdio://` — 标准流（适合命令行项目）
- `redis://127.0.0.1:6379` — Redis Stream（需要 Redis ≥ 5.0，适合分布式）
- `kafka://localhost:9092` — Kafka（适合大消息量分布式，可替换为 [Redpanda](https://redpanda.com)）

**各后端详细配置**：

```toml
[stream.file]
connect = { create_file = "CreateIfNotExists" }

[stream.stdio]
connect = { loopback = false }

[stream.redis]
connect = { db = 0, username = "user", password = "passwd" }

[stream.kafka]
connect = { sasl_options = { mechanism = "Plain", username = "user", password = "passwd" } }
```

### 发送消息

`StreamPlugin` 注册 `Producer` 组件。JSON 格式需启用 `json` feature。

### 消费消息 — `#[stream_listener]`

```rust
use summer_stream::stream_listener;
use summer::extractor::{Component, Config};

#[stream_listener("topic_name")]
async fn consumer(
    Component(db): Component<ConnectPool>,
    Config(conf): Config<CustomConfig>,
) {
    // 处理消息
}
```

### 提取配置

```rust
#[derive(Debug, Clone, Configurable, Deserialize)]
#[config_prefix = "custom"]
struct CustomConfig {
    a: u32,
    b: bool,
}

#[stream_listener("topic")]
async fn config_consumer(Config(conf): Config<CustomConfig>) {
    println!("a={}, b={}", conf.a, conf.b)
}
```

TOML：
```toml
[custom]
a = 1
b = true
```

完整示例：
- [stream-file-example](https://github.com/summer-rs/summer-rs/tree/master/examples/stream/stream-file-example)
- [stream-redis-example](https://github.com/summer-rs/summer-rs/tree/master/examples/stream/stream-redis-example)
- [stream-kafka-example](https://github.com/summer-rs/summer-rs/tree/master/examples/stream/stream-kafka-example)

> 注意：Stream 示例通常有多个二进制（producer / consumer），需 `cargo run --bin producer` / `--bin consumer`。运行前可能需要 `docker compose up` 启动 Kafka 等依赖。

