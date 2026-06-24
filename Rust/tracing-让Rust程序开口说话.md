## 你被这种场景折磨过吗？

**场景 A：** 生产环境出了 bug，你往代码里插满了 `println!`，重新部署，跑一遍，看日志——然后发现打了满屏的时间戳和消息，但完全看不出**哪个请求对应哪条日志**。

```
2026-06-24 10:01:23 连接数据库成功
2026-06-24 10:01:23 收到请求
2026-06-24 10:01:23 收到请求
2026-06-24 10:01:23 查询用户数据...
2026-06-24 10:01:23 查询用户数据...
2026-06-24 10:01:23 数据库返回结果
2026-06-24 10:01:23 数据库返回结果
```

两个请求的日志像两团毛线一样缠在一起。你分不清哪个"查询成功"对应哪个"收到请求"。

**场景 B：** `println!` 打出来的只有一句话。你看到"解析配置文件失败"，但你想知道——**是哪个函数调用的？参数是什么？当时的环境变量是什么？** 全都没有，就一句话。

---

如果你经历过上面任何一种情况，你需要的不是更会**打日志**，而是一个更好的**诊断模型**。

前两篇我们聊了怎么处理错误（`Result` / `?`）和怎么区分错误类型（`thiserror` / `anyhow`）。这一篇讲的是第三步：

> **错误发生了 → 你知道了是什么错 → 你怎么追查到底哪里出了问题？**

答案就是 `tracing`。

---

## 一句话先记住

| 传统日志 (log crate) | tracing |
|----------------------|---------|
| 平面的一行文字 | 带层级关系的事件树 |
| 无法关联同一个请求的多条日志 | `Span` 自动给同域日志分组 |
| 只有"谁打的"（target） | 还有"在哪个上下文打的"（span + field） |
| 结构化靠手拼 JSON | 原生结构化字段 |

> **log 是贴便签，tracing 是文件夹归档。**

---

## tracing 的核心：Event 和 Span

一共就两个概念：

### Event：一条日志

就是你熟悉的 `println!` 替代品：

```rust
use tracing::{info, warn, error};

fn process() {
    info!(user_id = 42, "开始处理用户数据");
    // 输出：[INFO] process: 开始处理用户数据 user_id=42

    let result: Result<(), String> = Err("超时".into());
    if let Err(e) = result {
        warn!(错误 = %e, "处理失败，准备重试");
        // 输出：[WARN] process: 处理失败，准备重试 错误=超时
    }
}
```

和 `println!` 的代码量差不多，但写法上有三个关键区别：

1. **结构化的键值对**：`user_id = 42` 而不是 `format!("用户 {}", id)`
2. **日志级别**：`info!` / `warn!` / `error!` / `debug!` / `trace!`
3. **自动记录代码位置**：文件、行号、模块路径

### Span：一段有头有尾的作用域

`Span` 才是 `tracing` 的灵魂。它像一对 `{ }` 大括号——进入时创建，退出时销毁，期间产生的所有 Event 自动挂在这个 Span 下面。

```rust
use tracing::{span, info, Level};

fn handle_request(id: u32) {
    let span = span!(Level::INFO, "handle_request", request_id = id);
    let _guard = span.enter();  // 进入 span

    info!("开始处理");

    // 在这个 span 内的所有 info! 都会带上 request_id
    // 输出：[handle_request request_id=42] 开始处理
}
```

这个 `_guard` 离开作用域时，span 自动关闭，`request_id` 消失。

---

## 快速上手：从 println! 到 tracing

### 第一步：安装

需要在 `Cargo.toml` 中加两个依赖：

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```

### 第二步：初始化订阅者

`tracing` 本身不输出日志——它产生事件，你告诉它**把事件输出到哪里**。

```rust
use tracing_subscriber::fmt;

// 最简单的配置：打印到终端
fn main() {
    fmt::init();

    // 现在 tracing 的所有宏都会输出到 stderr
    tracing::info!("应用启动");
}
```

### 第三步：替换 println!

```rust
use tracing::{info, warn};
use tracing_subscriber::fmt;

fn main() {
    fmt::init();

    let user = "alice";
    info!(user, "用户登录成功");

    let err_code = 403;
    warn!(user, err_code, "权限不足");
}
```

输出：

```
2026-06-24T10:01:23.123Z INFO  main: 用户登录成功 user="alice"
2026-06-24T10:01:23.124Z WARN  main: 权限不足 user="alice" err_code=403
```

每一行都带了时间戳、级别、模块路径、结构化的字段。**什么都不用手拼。**

---

## 核心能力一：Span——给日志加"文件夹"

### 嵌套 Span

看一个实际的 Web 请求场景：

```rust
use tracing::{span, info, warn, Level};

fn query_user(db: &str) {
    let span = span!(Level::INFO, "query_user", db = db);
    let _g = span.enter();
    info!("开始查询");
    // ... 查询逻辑
    info!("查询完成");
}

fn handle_request(id: u32) {
    let span = span!(Level::INFO, "handle_request", request_id = id);
    let _g = span.enter();

    info!("收到请求");
    query_user("users_db");
    info!("请求处理完成");
}

fn main() {
    tracing_subscriber::fmt::init();
    handle_request(42);
}
```

输出会像这样（结构简化，实际单行）：

```
 INFO handle_request{request_id=42}: 收到请求
 INFO handle_request{request_id=42}:query_user{db="users_db"}: 开始查询
 INFO handle_request{request_id=42}:query_user{db="users_db"}: 查询完成
 INFO handle_request{request_id=42}: 请求处理完成
```

**你看出来了吗？** 每条日志都自动"属于"它的上层 span。`query_user` 的日志前面带着 `handle_request{request_id=42}:query_user{db="users_db"}`——就像文件路径一样，记录了这条日志的完整**调用上下文**。

在 `log` crate 里，你想打出这个效果得手动拼字符串。在 `tracing` 里，**这是自动的**。

### #[instrument]——零侵入追踪

每次写 `let span = ...; let _g = span.enter();` 也挺累的。`#[instrument]` 帮你自动完成：

```rust
use tracing::info;
use tracing::instrument;

#[instrument]
fn process_order(order_id: u64, user: &str) {
    info!("处理订单");
    // 等价于自动创建 span!(Level::INFO, "process_order", order_id, user)
}
```

`#[instrument]` 会：
1. 自动创建以函数名命名的 span
2. 自动捕获所有参数作为 span 的字段
3. 自动在函数入口进入 span、退出时关闭 span

更进阶的用法：

```rust
#[instrument(skip(password))]  // 跳过敏感参数
fn login(user: &str, password: &str) {
    info!("登录中");
}

#[instrument(fields(request_id = %uuid::Uuid::new_v4()))]  // 自定义字段
fn handle_http() {
    info!("HTTP 请求到达");
}
```

---

## 核心能力二：灵活的输出配置

`fmt::init()` 只是默认配置。生产环境你需要更多控制：

### 1. 按模块过滤级别

```rust
use tracing_subscriber::fmt;
use tracing_subscriber::filter::EnvFilter;

fn main() {
    // 解析 RUST_LOG 环境变量
    // 用法：RUST_LOG=info,crate::module=debug ./app
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));

    fmt().with_env_filter(filter).init();

    tracing::info!("应用启动了");
}
```

这样可以通过环境变量控制日志级别，而不改代码：

```bash
# 只看 info 及以上
RUST_LOG=info ./app

# 看 myapp 模块的 debug 日志，其他模块只看 warn
RUST_LOG=warn,myapp=debug ./app

# 全部 trace 级别（最详细）
RUST_LOG=trace ./app
```

### 2. JSON 输出——给日志分析系统

```rust
use tracing_subscriber::fmt;

fn main() {
    // JSON 格式输出，每行一个 JSON 对象
    fmt()
        .json()
        .with_target(true)
        .with_current_span(true)
        .init();

    tracing::info!("服务器启动");
}
```

输出一行是一个合法的 JSON：

```json
{"timestamp":"2026-06-24T10:01:23.123Z","level":"INFO","fields":{"message":"服务器启动"},"target":"app","span":{"name":"main"}}
```

在 ELK、Datadog、 Loki 等日志系统里，这种格式可以直接解析。

### 3. 写入文件

```rust
use tracing_subscriber::fmt;
use std::fs::OpenOptions;

fn main() {
    let file = OpenOptions::new()
        .create(true).append(true)
        .open("app.log").unwrap();

    // 同时写入文件和控制台
    fmt()
        .with_writer(file)
        .with_ansi(false) // 文件里不输出 ANSI 颜色码
        .init();
}
```

> 生产环境更推荐用 `tracing-appender`（支持滚动日志），稍后会在彩蛋中展示。

---

## 实战：tracing 与错误处理集成

前两篇我们花了大量篇幅讲怎么定义和传播错误。现在让 **tracing 把错误的"所以然"记录下来**。

### 基础：在错误发生时打日志

```rust
use tracing::{error, instrument};
use std::fs;

#[instrument]
fn read_config(path: &str) -> Result<String, std::io::Error> {
    let content = fs::read_to_string(path)?;
    Ok(content)
}

fn main() {
    tracing_subscriber::fmt::init();

    if let Err(e) = read_config("/not/exist.toml") {
        error!(error = %e, "配置加载失败");
    }
}
```

输出会包含 `read_config(path="/not/exist.toml")` 的 span 信息——你不仅能看见"配置加载失败"，还能看见**传入了什么参数**。这在排查问题时价值巨大。

### 进阶：用 tracing-error 把 span 链注入错误

这是 `tracing` 和错误处理结合的精髓。

```rust
use tracing::{info, instrument};
use tracing_error::InstrumentError;
use anyhow::Context;

#[instrument]
fn step1() -> Result<(), anyhow::Error> {
    info!("执行步骤一");
    Ok(())
}

#[instrument]
fn step2() -> Result<(), anyhow::Error> {
    info!("执行步骤二");
    Err(anyhow::anyhow!("步骤二出错了"))
}

#[instrument]
fn run_pipeline() -> Result<(), anyhow::Error> {
    step1()?;
    step2()?;
    Ok(())
}

fn main() {
    // 初始化 tracing-error
    tracing_error::ErrorLayer::fmt().init();

    tracing_subscriber::fmt()
        .with_env_filter("info")
        .init();

    if let Err(e) = run_pipeline().context("流水线执行失败") {
        // 错误信息会包含完整的 span 调用链
        tracing::error!("{}", e);
        // 输出类似：
        // Error: 流水线执行失败
        // Caused by: 步骤二出错了
        //     with span_context: run_pipeline -> step2
        //     at src/main.rs:XX
    }
}
```

`tracing-error` 会把 span 层级信息编码到错误链中。当你拿到一个 `anyhow::Error` 时，不仅能看到错误消息，还能看到这条错误**经过了哪些函数**。

---

## 进阶技巧：不要做的事

### ❌ 在 println! 旁边混用 tracing

过渡期可以，但不要长期混用。`println!` 绕过所有 tracing 配置：
- 不受 `RUST_LOG` 过滤控制
- 不输出 JSON 格式
- 不带 span 上下文

### ❌ 在关键路径上打 debug! / trace!

`trace!` 和 `debug!` 在生产环境默认关闭。但如果你在循环里打了 `trace!`，即使不输出，参数的求值开销仍在。

安全的写法是使用 **lazy 求值**：

```rust
// ❌ 即使 level 是 info，format! 也会执行
debug!("计算结果：{}", expensive_calc());

// ✅ 使用 ? 或结构化字段
debug!(result = ?expensive_calc(), "计算完成");
```

### ❌ 把敏感信息写进 span 字段

Span 字段会被捕获并持久化到 span 的生命周期里。如果包含密码、token 等敏感信息，用 `#[instrument(skip(password))]` 跳过。

---

## 对比总结：tracing vs log vs println!

| 能力 | println! | log crate | tracing |
|------|----------|-----------|---------|
| 日志级别 | ❌ | ✅ | ✅ |
| 结构化字段 | ❌ 手拼 format! | ❌ 手拼 | ✅ 原生 |
| Span 上下文 | ❌ | ❌ | ✅ |
| 模块级别过滤 | ❌ | ✅ | ✅ |
| JSON 输出 | ❌ | 需额外库 | ✅ 原生 |
| 开销量级 | 中 | 低 | 中（但可控） |

---

## 彩蛋：带日志的生产级 CLI 模板

一个可以直接复制使用的完整示例，整合了之前文章的错误处理知识和今天的 tracing：

```rust
use anyhow::Context;
use tracing::{info, error, instrument, warn};
use tracing_subscriber::fmt;
use tracing_subscriber::filter::EnvFilter;
use tracing_appender::rolling;

#[instrument]
fn load_config(path: &str) -> anyhow::Result<String> {
    info!("读取配置文件");
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("读取配置文件失败：{}", path))?;
    info!(文件大小 = content.len(), "配置加载成功");
    Ok(content)
}

#[instrument]
fn validate_config(raw: &str) -> anyhow::Result<()> {
    warn!(长度 = raw.len(), "配置校验——当前使用简化逻辑");

    if raw.is_empty() {
        anyhow::bail!("配置文件不能为空");
    }

    info!("校验通过");
    Ok(())
}

#[instrument]
fn run_app(config_path: &str) -> anyhow::Result<()> {
    let raw = load_config(config_path)?;
    validate_config(&raw)?;
    info!("应用启动完成");
    Ok(())
}

fn main() -> anyhow::Result<()> {
    // 生产环境：日志同时写入 rolling 文件和终端
    let file_appender = rolling::daily("logs", "app.log");
    let (non_blocking, _guard) = tracing_appender::non_blocking(file_appender);

    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));

    fmt()
        .with_env_filter(filter)
        .with_writer(non_blocking)
        .with_ansi(false) // 文件输出不需要颜色
        .init();

    // 如果不指定 config_path，让它自然出错
    let config_path = std::env::var("CONFIG_PATH")
        .unwrap_or_else(|_| "config.toml".to_string());

    if let Err(e) = run_app(&config_path) {
        error!("应用异常退出：{:?}", e);
        // 关键：返回 Err 让进程以非零状态退出
        return Err(e);
    }

    Ok(())
}
```

使用方式：

```bash
# 默认 info 级别
CONFIG_PATH=myapp.toml cargo run

# 想看更详细的日志
RUST_LOG=debug CONFIG_PATH=myapp.toml cargo run

# 只看某个模块的 debug
RUST_LOG=warn,myapp=debug CONFIG_PATH=myapp.toml cargo run

# 日志会同时输出到终端和 logs/app.log.YYYY-MM-DD
```

不再需要 `println!`。不再需要手拼日志字符串。不再需要 grep 半天找一条请求的完整链路。

---

## 写在最后

前两篇我们解决了 Rust 错误的"分类"和"传播"，这一篇解决了"诊断"。

三篇合在一起，就构成了 Rust 生产级程序的**错误处理全链路**：

> **thiserror 定义错误 → anyhow 传播错误 → tracing 追踪错误**

这不仅是工具链的堆叠，而是一种思维方式的升级——**不再把错误当成"意外"，而是把它当成程序运行状态的一部分，可观测、可追踪、可诊断。**

三篇都读过之后，你已经掌握了 Rust 生产开发中最重要的几个环节。

---

*如果这篇文章对你有帮助，欢迎转发给也在写 Rust 的朋友 👊*
