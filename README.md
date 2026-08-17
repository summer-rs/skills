# summer-rs Skill

为 CodeBuddy 提供 summer-rs 框架知识的 AI Skill。激活后，AI 能正确使用 summer-rs 的核心概念、插件系统、配置管理、依赖注入以及全部官方插件。

## 安装方式

### 方式一：复制到项目（推荐，团队共享）

将整个 `summer-rs/` 目录复制到项目的 `.codebuddy/skills/` 下：

```bash
# 在项目根目录执行
mkdir -p .codebuddy/skills
cp -r skills/summer-rs .codebuddy/skills/summer-rs
```

AI 在该项目中会自动加载此 skill。

### 方式二：用户级安装（所有项目可用）

复制到用户目录：

```bash
mkdir -p ~/.codebuddy/skills
cp -r skills/summer-rs ~/.codebuddy/skills/summer-rs
```

AI 在所有项目中都会自动加载此 skill。

### 方式三：打包分发

使用 CodeBuddy 的打包工具生成 `.zip` 文件：

```bash
# 在 CodeBuddy 环境中执行（需要内置的 package_skill.py 脚本）
# 打包后发给其他人，解压到 .codebuddy/skills/ 即可使用
```

## 文件结构

```
summer-rs/
├── SKILL.md                         # 核心入口，框架概念、快速上手、插件列表
├── README.md                        # 本文件
└── references/                      # 按需加载的详细文档
    ├── plugins.md                   # 插件系统、Component 宏、配置系统内部机制
    ├── web_plugin.md                # summer-web：路由、提取器、中间件、OpenAPI
    ├── data_plugins.md              # SQLx、PostgreSQL、SeaORM、Redis
    ├── job_stream_plugins.md        # 定时任务、消息流
    ├── misc_plugins.md              # Mail、gRPC、OpenTelemetry、Nacos、xxl-job 等
    ├── recipes.md                   # 常用模式实例（CRUD、认证、缓存、测试等）
    └── troubleshooting.md           # 常见错误排查
```

## 触发条件

当用户提到以下内容时，AI 自动激活此 skill：

- 创建 summer-rs 项目、添加框架依赖
- 使用 `summer-web` / `summer-sqlx` / `summer-redis` 等插件
- 编写 `#[get]` / `#[post]` 路由、`#[component]` 工厂、`#[derive(Service)]` DI
- 配置 TOML、环境变量插值、多环境合并
- 定时任务、消息流、gRPC、Nacos、xxl-job 等场景
- 排查 summer-rs 运行或构建错误

