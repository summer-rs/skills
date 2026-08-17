# summer-rs Troubleshooting Guide

Common errors when working with summer-rs and how to resolve them.

---

## Config Errors

### `Config X not found`

**Cause**: The TOML configuration section for plugin/component X is missing or misnamed.

**Diagnosis**:
- Check that the `#[config_prefix]` value matches the TOML section name exactly
- Verify the config file is at `./config/app.toml` (or the environment variant)
- Confirm `SUMMER_ENV` is set correctly if using environment-specific configs

**Solution**:
```toml
# If #[config_prefix = "my-db"], add to config/app.toml:
[my-db]
host = "localhost"
port = 5432
```

### Config deserialization error

**Cause**: TOML value types don't match the Rust struct field types.

**Diagnosis**: The error message will indicate which field failed to deserialize.

**Solution**: Ensure types match. Common mismatches:
- Rust `u16` but TOML number is `100000` (overflow)
- Rust `bool` but TOML has `"true"` (string vs bool)
- Rust `Vec<String>` but TOML has single string

---

## Component Errors

### `Component X not found`

**Cause**: A `#[component]` factory or handler depends on component X, but X was never registered.

**Diagnosis**:
- Check if X's factory function has `#[component]` annotation
- If X is registered manually, verify `app.add_component(x)` is called in the Plugin's `build` method
- Check plugin loading order — the plugin that registers X must be added to `App` before the plugin that needs it

**Solution**: Either add `#[component]` to X's factory, or call `app.add_component(x)` in the dependent plugin's `build`.

### `component was already added`

**Cause**: Two `#[component]` factories or manual registrations produce the same Rust type.

**Diagnosis**: The DI container identifies components by `TypeId`. Two factories returning the same type conflict.

**Solution**: Use the NewType pattern:
```rust
#[derive(Clone)]
struct PrimaryDb(DbConnection);

#[derive(Clone)]
struct SecondaryDb(DbConnection);

#[component(name = "PrimaryDatabase")]
fn create_primary_db(Config(config): Config<PrimaryDbConfig>) -> PrimaryDb {
    PrimaryDb(DbConnection::new(&config))
}
```

### `plugin was already added`

**Cause**: Two plugins with the same `name()` are added to the App.

**Diagnosis**: By default `name()` returns `std::any::type_name::<Self>()`. Two `#[component]` macros without explicit names produce the same plugin name.

**Solution**: Use `#[component(name = "UniqueName")]` to distinguish.

---

## Dependency Errors

### `Cyclic dependency detected`

**Cause**: Component A depends on B, B depends on A (directly or transitively).

**Diagnosis**: Trace the `Component<T>` parameters in `#[component]` factory functions and `#[inject(component)]` in `Service` structs.

**Solutions** (in order of preference):
1. **Refactor**: Split the cycle into three components with a shared interface
2. **LazyComponent**: Wrap one side in `LazyComponent<T>`
   ```rust
   #[derive(Clone, Service)]
   struct ServiceA {
       service_b: LazyComponent<ServiceB>,
   }
   ```
3. **Event-driven**: Use the event system to break the dependency

### Plugin dependency not met

**Cause**: Plugin A declares `dependencies() -> vec!["some_plugin"]` but that plugin was never added.

**Solution**: Ensure all declared dependencies are added to `App::new()` before the dependent plugin.

---

## Web Plugin Errors

### Route not found (404 despite having a handler)

**Common causes**:
- Forgot to `#[auto_config(WebConfigurator)]` on `main()`
- Handler function is in a module not imported into scope (Rust dead-code elimination)
- Forgot `use summer_web::get;` (or other method macros)
- Route prefix mismatch: check `[web] global_prefix = "api"` in config
- Manual registration: forgot `app.add_router(router())`

### Extractor errors in middleware

**Cause**: Axum middleware functions have specific extractor rules. Not all summer-web extractors work in middleware context.

**Solution**: Middleware extractors must implement Axum's `FromRequestParts`. The `Component` extractor (`summer_web::extractor::Component`) implements this and works in middleware. For complex extraction, do it inside the handler instead.

### `ConnectInfo` not available

**Cause**: `connect_info` is not enabled in Web config.

**Solution**:
```toml
[web]
connect_info = true
```

---

## Database Errors

### SQLx connection fails

**Common causes**:
- URI format incorrect: `postgres://user:password@host:port/dbname` (note: `postgres://` not `postgresql://`)
- Database doesn't exist: create it first
- Network: ensure the database server is reachable
- Credentials: verify user/password

### SQLx compile-time check failing

**Cause**: SQLx by default verifies SQL queries at compile time against a running database.

**Solutions**:
- Set `DATABASE_URL` environment variable before `cargo build`
- Use `sqlx::query()` instead of `sqlx::query!("...")` to skip compile-time checks
- Use `sqlx::query_as::<_, T>("...")` for runtime-checked queries

### SeaORM - entity not found

**Cause**: Entity code was generated by `sea-orm-cli` but the module isn't properly imported.

**Solution**: Ensure the generated entity module is included in `src/` and properly `mod` declared.

---

## Runtime Errors

### Application hangs on startup

**Common causes**:
- Database connection timeout (check `acquire_timeout` in config)
- Redis server unreachable
- Nacos server unreachable when `enable_config = true`

**Solution**: Check connectivity to external services by inspecting the last plugin that tried to build (look at the logs).

### Port already in use

**Cause**: Another process is using the configured port.

**Solution**: Change `[web] port` in config, or kill the existing process.

### `SUMMER_ENV` not taking effect

**Cause**: The env var is set but the config file is not found.

**Diagnosis**: Framework looks for `./config/app-{env}.toml` relative to the working directory.

**Solution**: Run the application from the directory containing `config/`, ensure file is named correctly (e.g., `app-dev.toml` for `SUMMER_ENV=dev`).

---

## Build Errors

### `#[component]` compilation fails

**Common issues**:
- Factory function is not `pub` or is in a module not reachable from the crate root
- Return type doesn't implement `Clone + Send + Sync + 'static`
- Config type missing `Configurable` derive or has wrong prefix
- `#[inject("name")]` references a name that doesn't exist

### `#[derive(Service)]` compilation fails

**Common issues**:
- Struct fields with `#[inject(component)]` use types not registered as components
- Using `#[inject(config)]` on a type that doesn't implement `Configurable`
- `LazyComponent<T>` where T doesn't implement `Service`

### `inventory` / linker errors about duplicate symbols

**Cause**: Two `#[component]` macros produce the same plugin type identifier. This is a linker-level conflict.

**Solution**: Ensure each `#[component]` factory has a unique `name` parameter when they produce the same component type. Even better, use NewType wrappers.

---

## Testing Errors

### `MockServer` not found as component

**Cause**: `MockWebPlugin` was not added, or `build()` was called without it.

**Solution**:
```rust
App::new()
    .add_plugin(MockWebPlugin)  // Must be added
    .add_router(router())
    .build()
    .await?;
```

### Test server doesn't respond as expected

**Diagnosis**: `MockWebPlugin` reuses the same routing logic as `WebPlugin`. Check:
- Routes are registered correctly via `add_router()`
- Middleware applies as expected
- `#[auto_config]` is not used (not needed for test, manual router is clearer)

---

## Environment-Specific Issues

### Running examples from the repo

Always `cd` into the example directory (the one with `Cargo.toml` and `config/`) before `cargo run`. The framework resolves config paths relative to the working directory.

### Docker / containerized deployment

- Config files must be accessible inside the container at `./config/app.toml`
- Database URIs should use container networking (e.g., `postgres://user:pass@db:5432/db`)
- Set `SUMMER_ENV` in the container environment for profile-specific configs

