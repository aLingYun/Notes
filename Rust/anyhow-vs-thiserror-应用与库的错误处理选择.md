## 你被这两种感觉折磨过吗？

**场景 A：** 写了一个库函数，想把所有可能的错误列清楚，于是手搓了一个 `enum`：

```rust
enum MyError {
    Io(std::io::Error),
    Parse(serde_json::Error),
    Net(reqwest::Error),
}
```

写到第五个变体，心态崩了——这手动 `impl Display` + `impl From` 也太累了吧？

**场景 B：** 写一个 CLI 工具，调了一堆库，每个函数返回不同的错误类型：

```rust
fn step1() -> Result<(), io::Error> { ... }
fn step2() -> Result<(), serde_json::Error> { ... }
fn step3() -> Result<(), reqwest::Error> { ... }
```

你想串起来，但 `?` 的类型不匹配，编译器报错。于是你开始写 `map_err` 大战……

---

如果你经历过上面任何一种情况，今天这篇文章就是为你写的。

上篇我们讲了 Rust 错误处理的核心工具链（`Result`、`?`、`thiserror`）。但那个表格里有一句话只说了一半：

> **anyhow 用于应用层，thiserror 用于库**

今天就把这句展开，帮你彻底搞懂 **anyhow 和 thiserror 各自解决什么问题、什么时候用哪一个、怎么配合用**。

---

## 一句话先记住

| Crate | 最佳场景 | 核心价值 |
|-------|----------|----------|
| **thiserror** | 写库/底层模块 | 用 derive 宏帮你定义**明确、可枚举**的错误类型 |
| **anyhow** | 写应用/CLI/脚本 | 让你在**不同错误类型之间自由转换**，并提供上下文 |

> 一个写给别人用，一个写给自己用。

---

## 场景一：thiserror——给 Rustaceans 的礼物

上篇我们只简单用了一下 `#[derive(Error)]`，其实它的全貌更优雅。

### 三行搞定一个完整错误类型

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("网络请求失败：{0}")]
    Network(#[from] std::io::Error),

    #[error("数据解析失败：{0}")]
    Parse(#[from] serde_json::Error),
}
```

就这么三块内容：

1. **`#[derive(Error)]`**——自动实现 `std::error::Error` trait
2. **`#[error("...")]`**——定义 `Display` 输出的文本，支持的占位符：
   - `{0}` —— 元组字段的 Display
   - `{field}` —— 具名字段的 Display
   - `{field:?}` —— 具名字段的 Debug
3. **`#[from]`**——自动生成 `From<源错误> for AppError`，`?` 操作符直接能用

### 不用 thiserror 的话，要写多少代码？

```rust
// 手写一个变体
#[derive(Debug)]
pub enum AppError {
    Network(std::io::Error),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::Network(e) => write!(f, "网络请求失败：{}", e),
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::Network(e) => Some(e),
        }
    }
}

impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self {
        AppError::Network(e)
    }
}
```

**三行 vs 二十行**。这还不是一个复杂场景。

这是 `thiserror` 最直接的吸引力——**减少样板代码，且不牺牲错误类型的明确性**。

---

## 场景二：anyhow——应用层的瑞士军刀

先看一个典型应用代码的痛点：

```rust
// 调用三个函数，返回三种错误类型
fn deploy_app() -> Result<String, ???> {
    let config = std::fs::read_to_string("deploy.toml")?; // io::Error
    let parsed: toml::Value = toml::from_str(&config)?;   // toml::de::Error
    let db_url = std::env::var("DATABASE_URL")?;          // VarError
    Ok(db_url)
}
```

这三个 `?` 后面对应三种不同的错误类型。如果你用 `thiserror`，需要定义一个包含三种变体的枚举，再写三个 `#[from]`。不麻烦，但——**在应用代码里，你根本不在乎具体是哪一种错误**，你只想：

1. 跳过不匹配的类型继续写
2. 出了错知道**在哪个步骤**出的
3. 给用户一句人能看懂的话

这就是 `anyhow` 的场景。

### anyhow::Result

```rust
use anyhow::Result;  // 替代 std::result::Result<T, Box<dyn Error>>

fn deploy_app() -> Result<String> {
    let config = std::fs::read_to_string("deploy.toml")?;
    let parsed: toml::Value = toml::from_str(&config)?;
    let db_url = std::env::var("DATABASE_URL")?;
    Ok(db_url)
}
```

`anyhow::Result<T>` 实际上是 `std::result::Result<T, anyhow::Error>`。`anyhow::Error` 可以容纳**任何实现了 `std::error::Error` 的类型**——Rust 生态里99%的错误都满足这个条件。

**三个 `?`，零个 `map_err`，编译通过。**

### 给错误添加上下文

这才是 `anyhow` 最实用的能力。

```rust
use anyhow::{Context, Result};

fn deploy_app() -> Result<String> {
    let config = std::fs::read_to_string("deploy.toml")
        .context("读取部署配置文件失败")?;

    let parsed: toml::Value = toml::from_str(&config)
        .context("解析 TOML 配置失败")?;

    let db_url = std::env::var("DATABASE_URL")
        .context("获取环境变量 DATABASE_URL 失败")?;

    Ok(db_url)
}
```

错误链会变成类似这样的输出：

```
Error: 读取部署配置文件失败
Caused by:
    No such file or directory (os error 2)
```

或者用 `{:#}` 格式化：

```
读取部署配置文件失败: No such file or directory (os error 2)
```

每一个 `.context()` 就是一层**解释**。原始错误（"No such file"）告诉系统发生了什么，`context` 告诉运维**这个错误发生在哪个业务步骤**。两者都保留，不是覆盖。

### anyhow! 宏：快速创建临时错误

```rust
use anyhow::{anyhow, Result};

fn validate_env() -> Result<()> {
    if std::env::var("DATABASE_URL").is_err() {
        return Err(anyhow!("缺少环境变量 DATABASE_URL"));
    }
    Ok(())
}
```

不用定义自定义错误类型，不需要 `#[error]` 宏，**临时起意的一条错误消息**。适合验证逻辑、快速脚本、原型开发。

还有 `bail!` 宏，是 `return Err(anyhow!(...))` 的缩写：

```rust
fn validate_env() -> Result<()> {
    bail!("缺少环境变量 DATABASE_URL");
}
```

---

## 两者对比：这么选就对了

| 维度 | thiserror | anyhow |
|------|-----------|--------|
| 目标用户 | 库的调用者 | 你自己 / 团队 |
| 错误信息 | 精确的类型枚举 | 灵活的上下文说明 |
| 匹配处理 | ✅ match 具体错误变体 | ❌ 只能 downcast，但通常不需要 |
| 变更多少 | 加一个错误变体需要改代码 | 加一个 .context() 不改变签名 |
| 典型位置 | lib.rs / 模块内部 | main.rs / CLI / 业务流程 |
| 组合使用 | 库定义错误 → 应用用 anyhow 包装 |

### 什么时候选 thiserror？

- 你在写一个**库**，别人会调用你的代码
- 调用者需要根据不同的错误做不同处理（是重试还是放弃、是网络问题还是数据问题）
- 错误类型是 API 契约的一部分

### 什么时候选 anyhow？

- 你在写 **CLI 工具、Web 服务、脚本**
- 你不在乎具体是哪一种错误，只想知道**出了什么问题、在哪一步出的**
- 快速验证想法，不想在错误类型上花时间

---

## 最佳实践：库用 thiserror，应用用 anyhow

这是 Rust 生态的**黄金组合**。

### 分层架构中的错误处理

```
┌──────────────────────────────────┐
│         应用层 (main.rs)         │  ← anyhow::Result + .context()
│  CLI / Web 服务 / 业务流程       │
├──────────────────────────────────┤
│       领域层 (domain/)           │  ← thiserror::Error 枚举
│  业务逻辑、验证规则               │
├──────────────────────────────────┤
│       基础设施层 (infra/)        │  ← 第三方错误透传
│  文件IO、网络请求、数据库          │
└──────────────────────────────────┘
```

### 实战：一个完整的服务加载器

先看库的定义，用 `thiserror` 列出调用者需要关心的错误：

```rust
// lib.rs — 错误是API的一部分，用 thiserror
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("服务 {name} 连接超时")]
    Timeout { name: String },

    #[error("配置文件无效：{detail}")]
    InvalidConfig { detail: String },

    #[error("初始化失败：{0}")]
    InitFailed(#[from] std::io::Error),
}

#[derive(Debug)]
pub struct ServiceState {
    pub name: String,
}

pub fn load_service(path: &str) -> Result<ServiceState, ServiceError> {
    if path.is_empty() {
        return Err(ServiceError::InvalidConfig {
            detail: "路径不能为空".to_string(),
        });
    }
    let _config = std::fs::read_to_string(path)?;  // io::Error → ServiceError
    Ok(ServiceState {
        name: "database".to_string(),
    })
}
```

调用者可以根据错误类型做不同处理：

```rust
// 调用端 — 需要区分重试 vs 报错
use mylib::ServiceError;

match load_service("/etc/myapp/config.toml") {
    Ok(state) => start_server(state),
    Err(ServiceError::Timeout { name }) => {
        // 超时 — 等几秒重试
        retry_later(name);
    }
    Err(ServiceError::InvalidConfig { detail }) => {
        // 配置无效 — 直接退出，不需要重试
        eprintln!("配置错误：{}，请检查配置文件", detail);
        std::process::exit(1);
    }
    Err(e) => {
        // 其他错误 — 打印全部信息
        eprintln!("未知错误：{}", e);
    }
}
```

但如果这是**应用本身**的 main 函数，不需要区分——用 anyhow 更舒服：

```rust
// main.rs — 应用层，用 anyhow
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let state = mylib::load_service("/etc/myapp/config.toml")
        .context("加载服务失败")?;

    // 如果出错，输出类似：
    // Error: 加载服务失败
    // Caused by: 服务 database 连接超时

    start_server(state);
    Ok(())
}
```

`load_service` 返回 `ServiceError`（thiserror），但在 main 里 `.context()` 把它包成了 `anyhow::Error`。**类型从精确变灵活，信息从清晰变丰富**。

---

## 进阶技巧：装饰器模式

`anyhow` 的 `bail!` 和 `ensure!` 很适合做前置校验的**装饰器**：

```rust
use anyhow::{ensure, Result};

fn deploy(branch: &str, env: &str) -> Result<()> {
    ensure!(!branch.is_empty(), "分支名不能为空");
    ensure!(env == "staging" || env == "production", "环境必须是 staging 或 production");

    // 真正的部署逻辑
    Ok(())
}
```

不污染业务逻辑，校验和核心逻辑分离。

---

## 对比总结表格：如何选择

```
    ┌─────────────────────────────────────┐
    │          你写的是库还是应用？        │
    ├─────────────────┬───────────────────┤
    │     是库         │     是应用        │
    ├─────────────────┴───────────────────┤
    │        │                            │
    │        ▼                            ▼
    │  调用者需要 match 吗？      需要保留上下文吗？
    │   │         │              │         │
    │  是│         │否          是│         │否
    │   ▼         ▼             ▼         ▼
    │ thiserror    anyhow     anyhow  标准库(够用)
    │ (或者两者皆可)
    └─────────────────────────────────────┘
```

---

## 彩蛋：完整实战模板

一个可直接复制使用的**分层架构示例**：

```rust
// ============ lib.rs ============
use serde::Deserialize;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ConfigError {
    #[error("读取配置失败：{0}")]
    ReadFailed(#[from] std::io::Error),

    #[error("解析配置失败：{0}")]
    ParseFailed(#[from] toml::de::Error),

    #[error("缺少必要字段：{field}")]
    MissingField { field: &'static str },
}

#[derive(Deserialize, Debug)]
pub struct AppConfig {
    pub host: String,
    pub port: u16,
    pub database_url: String,
}

pub fn parse_config(path: &str) -> Result<AppConfig, ConfigError> {
    let content = std::fs::read_to_string(path)?;
    let config: AppConfig = toml::from_str(&content)?;

    if config.database_url.is_empty() {
        return Err(ConfigError::MissingField { field: "database_url" });
    }

    Ok(config)
}

// ============ main.rs ============
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = parse_config("app.toml")
        .context("初始化应用配置失败")?;

    println!("服务启动：{}:{}", config.host, config.port);

    // 启动 HTTP 服务等...
    Ok(())
}
```

---

## 写在最后

`thiserror` 和 `anyhow` 不是竞争关系，它们是 **Rust 错误处理的两个层次**：

- **`thiserror`** 让你把错误设计清楚——调用者能 match、能处理、能恢复
- **`anyhow`** 让你把错误串联好——在应用层一眼看到"哪一步出了什么问题"

用一句话记牢它们的定位：

> **库的错误是 API，要精确；应用的错误是诊断，要清晰。**

结合上篇学的 `Result` 和 `?`，你现在已经拥有了 Rust 生产环境错误处理的完整工具箱。下一篇想看什么？可以聊聊 tracing 日志、或者重试策略——你来选。

---

*如果这篇文章对你有帮助，欢迎转发给也在写 Rust 的朋友 👊*
