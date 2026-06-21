## 被这个报错折磨过吗？

```
thread 'main' panicked at 'called Option::unwrap() on a None value'
```

恭喜，你踩到了 Rust 最"卷"的地方——**错误处理**。

很多人学 Rust，到这一章就卡住了。`Result` 是什么？`?` 怎么用？为什么编译器一直报 warning？

不怪你。错误处理在大多数语言里都是"差不多就行"的领域，但 Rust 偏不——**它把错误当成一等公民，逼你必须认真面对**。

今天这篇文章，帮你彻底搞懂 Rust 错误处理的核心工具链。读完就能上手，用到生产环境。

---

## 先搞清楚：Rust 把错误分成了两类

这不是 Rust 任性，这是设计哲学。

| 类型 | 场景 | 怎么应对 |
|------|------|----------|
| **可恢复错误** `Result<T, E>` | 文件找不到、网络超时、解析失败 | 重试、降级、给用户提示 |
| **不可恢复错误** `panic!` | 程序走到了不该走到的地方 | 直接崩溃，别挣扎了 |

这个区分的价值在于：**你在代码里能看到所有可能的失败路径**。漏处理了一个错误？编译器直接拦住你。

---

## 从一个真实场景开始

假设你在写一个 CLI 工具，需要读取配置文件。

```rust
use std::fs;

fn read_config(path: &str) -> Result<String, std::io::Error> {
    fs::read_to_string(path)
}
```

就三行。

`fs::read_to_string` 的签名是 `fn read_to_string<P: AsRef<Path>>(path: P) -> Result<String, std::io::Error>`。如果文件不存在、权限不足、或者路径是个目录，都会返回 `Err`。

现在问题来了——**你拿到了 Result，然后呢？**

---

## 新手最常踩的坑：Result 被忽略了

```rust
fn main() {
    let content = fs::read_to_string("config.json");
    // 编译通过，但 content 是 Result<String, Error>
    // 你得到的只是一个错误值，不是字符串
}
```

这是 Rust 新手第一名错误——写了 `Result`，却忘了处理它。

正确写法：

```rust
use std::fs;
fn main() {
    match fs::read_to_string("config.json") {
        Ok(content) => println!("配置内容：{}", content),
        Err(e) => eprintln!("读取失败：{}", e),
    }
}
```

但 `match` 写多了真的很烦。Rust 有一个更优雅的方式。

---

## `?` 操作符：写少一点，错误传播交给它

`?` 是 Rust 错误处理最核心的语法糖，一句话理解：

> **如果这里是 `Ok`，解包出值继续；如果这里是 `Err`，直接把这个 `Err` 返回给调用者。**

### 单层使用

```rust
use std::fs;
use std::io;

fn read_config(path: &str) -> Result<String, io::Error> {
    let content = fs::read_to_string(path)?;  // Err 会直接返回
    Ok(content)
}
```

没有 match，没有嵌套，就一个 `?`。

### 链式调用，层层传播

这才是 `?` 真正发光的地方：

```rust
use std::fs;
use std::io;

fn load_and_parse(path: &str) -> Result<String, io::Error> {
    let content = fs::read_to_string(path)?;  // io::Error
    let trimmed = content.trim();              // 这是 &str，没有错误
    if trimmed.is_empty() {
        return Err(io::Error::new(
            io::ErrorKind::InvalidData,
            "配置文件为空",
        ));
    }
    Ok(trimmed.to_string())
}
```

每一步出错了，错误自动向外传，**调用者不需要知道中间发生了什么**。

这就是错误传播的美感。

---

> **小插曲：** 严格来说 `Option<T>` 不属于"错误处理"——它表示的是"值可能存在也可能不存在"，而不是"操作成功还是失败"。但因为 `?` 操作符同样适用于 `Option`，且在实际代码中 `Result` 和 `Option` 经常交替使用，所以放在一起讲。

## `Option<T>`：当"没有"本身就是答案

`Option<T>` 专门用来表示**值可能不存在**的情况：

```rust
pub enum Option<T> {
    None,       // 真的没有
    Some(T),    // 有这个值
}
```

典型的使用场景：

```rust
fn find_user(users: &[User], id: u32) -> Option<&User> {
    users.iter().find(|u| u.id == id)
}

let user = find_user(&users, 42);
match user {
    Some(u) => println!("找到用户：{}", u.name),
    None => println!("该用户不存在"),
}
```

### 链式操作，更优雅

```rust
let name = find_user(&users, 42)
    .map(|u| u.name.clone())          // 找到就取名字，没有就跳过
    .unwrap_or_else(|| "匿名用户".to_string());
```

这条链的含义：**找到用户 → 取名字 → 没有就默认"匿名用户"**。零嵌套，一气呵成。

---

## 生产级实践：自定义错误类型

应用复杂了，仅用标准库的 `io::Error` 就hold不住了。你需要区分不同来源的错误，给出有意义的提示。

用 `thiserror` 来定义错误类型（Rust 生态里最广泛使用的库）：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("配置文件不存在：{path}")]
    ConfigNotFound { path: String },

    #[error("JSON 解析失败：{0}")]
    ParseError(#[from] serde_json::Error),

    #[error("无效的配置值：{key}={value}")]
    InvalidConfig { key: String, value: String },
}
```

关键点：
- `#[error(...)]` 定义人类可读的错误消息，`{0}` / `{key}` 会在运行时被填充
- `#[from]` 标记的变体支持**自动类型转换**，`serde_json::Error` 自动变成 `AppError::ParseError`

实际使用：

```rust
fn parse_config(path: &str) -> Result<Config, AppError> {
    let content = fs::read_to_string(path)
        .map_err(|_| AppError::ConfigNotFound { path: path.to_string() })?;

    // 因为 ParseError 标记了 #[from]，? 会自动把 serde_json::Error 转成 AppError::ParseError
    let config: Config = serde_json::from_str(&content)?;

    Ok(config)
}
```

`map_err` 把一个错误类型转换成另一个，灵活组合。

### 错误链：追查根本原因

`#[from]` 的真正价值在于保留了**错误的因果链**：

```rust
use std::error::Error;

match parse_config("missing.json") {
    Ok(config) => println!("启动服务：{}:{}", config.host, config.port),
    Err(e) => {
        eprintln!("启动失败：{}", e);
        if let Some(cause) = e.source() {
            eprintln!("  根本原因：{}", cause);
        }
    }
}
```

`e.source()` 能拿到被 `#[from]` 包装的原始错误（需要 `use std::error::Error` 引入 trait），排查复杂问题时非常有用。

---

## 容易忽略的最佳实践

| 实践 | 说明 |
|------|------|
| **不要轻易用 `.unwrap()`** | 生产代码里，unwrap() 出错会直接 panic。测试或确定不会失败的地方才用 |
| **`?` 只能用于返回 Result 或 Option 的函数** | 类型必须匹配，这是 Rust 类型系统的保证 |
| **anyhow 用于应用层，thiserror 用于库** | anyhow::Result<T> 灵活；thiserror 给你可控的错误枚举 |
| **错误消息要给运维看** | "读取文件失败"不如"读取 /etc/app/config.json 失败：Permission denied" |
| **区分"我处理不了"和"我选择不处理"** | 传播错误是你的责任，推给调用者；捕获错误给用户友好提示是你的义务 |

---

## 写在最后

Rust 的错误处理，初看确实约束多——Result 要匹配，Option 要解包，错误要传播。

但当你真正习惯了这套系统之后，你会体会到它的价值：

**所有可能的失败路径，在代码里都是可见的。**

这不是负担，这是信任。编译器替你检查了那些你本来可能会忘记处理的边界情况。

错误不是程序的意外，它是 **API 的一部分**。

把错误设计好，你的 API 才真正好用。

---

**彩蛋：实战完整代码**

整合所有知识点，一个可直接使用的配置加载函数：

```rust
use std::fs;
use thiserror::Error;
use serde::Deserialize;

#[derive(Error, Debug)]
enum ConfigError {
    #[error("文件不存在：{path}")]
    NotFound { path: String },
    #[error("JSON 格式错误：{0}")]
    ParseError(#[from] serde_json::Error),
}

#[derive(Deserialize, Debug)]
pub struct Config {
    pub host: String,
    pub port: u16,
}

pub fn load_config(path: &str) -> Result<Config, ConfigError> {
    let raw = fs::read_to_string(path)
        .map_err(|_| ConfigError::NotFound { path: path.to_string() })?;

    let config: Config = serde_json::from_str(&raw)?;
    Ok(config)
}
```

拿这个当模板，改改错误类型，你的配置加载代码就已经是生产级了。

---

*如果这篇文章对你有帮助，欢迎转发给也在学 Rust 的朋友 👊*
