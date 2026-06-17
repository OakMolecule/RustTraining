# Rust `Option` 与 `Result` 要点

> **你将学到：** 惯用错误处理模式——`unwrap()` 的安全替代、`?` 传播、自定义错误类型，以及生产代码中何时用 `anyhow` vs `thiserror`。

- ```Option``` 与 ```Result``` 是惯用 Rust 的核心部分
- **`unwrap()` 的安全替代**：
```rust
// Option<T> safe alternatives
let value = opt.unwrap_or(default);              // Provide fallback value
let value = opt.unwrap_or_else(|| compute());    // Lazy computation for fallback
let value = opt.unwrap_or_default();             // Use Default trait implementation
let value = opt.expect("descriptive message");   // Only when panic is acceptable

// Result<T, E> safe alternatives  
let value = result.unwrap_or(fallback);          // Ignore error, use fallback
let value = result.unwrap_or_else(|e| handle(e)); // Handle error, return fallback
let value = result.unwrap_or_default();          // Use Default trait
```
- **用模式匹配显式控制**：
```rust
match some_option {
    Some(value) => println!("Got: {}", value),
    None => println!("No value found"),
}

match some_result {
    Ok(value) => process(value),
    Err(error) => log_error(error),
}
```
- **用 `?` 传播错误**：短路并向上冒泡错误
```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?; // Automatically returns error
    Ok(content.to_uppercase())
}
```
- **变换方法**：
    - `map()`：变换成功值 `Ok(T)` -> `Ok(U)` 或 `Some(T)` -> `Some(U)`
    - `map_err()`：变换错误类型 `Err(E)` -> `Err(F)`
    - `and_then()`：链接可能失败的操作
- **在自有 API 中使用**：优先 `Result<T, E>` 而非异常或错误码
- **参考**：[Option 文档](https://doc.rust-lang.org/std/option/enum.Option.html) | [Result 文档](https://doc.rust-lang.org/std/result/enum.Result.html)

# Rust 常见陷阱与调试技巧
- **借用问题**：新手最常见错误
    - "cannot borrow as mutable" -> 同时只允许一个可变引用
    - "borrowed value does not live long enough" -> 引用比所指数据活得更久
    - **修复**：用作用域 `{}` 限制引用生命周期，或必要时 clone 数据
- **缺少 Trait 实现**："method not found" 错误
    - **修复**：为常见 Trait 添加 `#[derive(Debug, Clone, PartialEq)]`
    - 用 `cargo check` 获得比 `cargo run` 更好的错误信息
- **Debug 模式下整数溢出**：Rust 在溢出时 panic
    - **修复**：用 `wrapping_add()`、`saturating_add()` 或 `checked_add()` 明确行为
- **`String` vs `&str` 混淆**：不同场景用不同类型
    - 字符串切片用 `&str`（借用），拥有字符串用 `String`
    - **修复**：用 `.to_string()` 或 `String::from()` 将 `&str` 转为 `String`
- **与借用检查器对抗**：不要试图绕过它
    - **修复**：重构代码以配合所有权规则，而非对抗
    - 复杂共享场景可考虑 `Rc<RefCell<T>>`（谨慎使用）

## 错误处理示例：好 vs 坏
```rust
// [ERROR] BAD: Can panic unexpectedly
fn bad_config_reader() -> String {
    let config = std::env::var("CONFIG_FILE").unwrap(); // Panic if not set!
    std::fs::read_to_string(config).unwrap()           // Panic if file missing!
}

// [OK] GOOD: Handles errors gracefully
fn good_config_reader() -> Result<String, ConfigError> {
    let config_path = std::env::var("CONFIG_FILE")
        .unwrap_or_else(|_| "default.conf".to_string()); // Fallback to default
    
    let content = std::fs::read_to_string(config_path)
        .map_err(ConfigError::FileRead)?;                // Convert and propagate error
    
    Ok(content)
}

// [OK] EVEN BETTER: With proper error types
use thiserror::Error;

#[derive(Error, Debug)]
enum ConfigError {
    #[error("Failed to read config file: {0}")]
    FileRead(#[from] std::io::Error),
    
    #[error("Invalid configuration: {message}")]
    Invalid { message: String },
}
```

下面分解发生了什么。`ConfigError` 只有**两个变体**——I/O 错误与校验错误。这是多数 module 的合适起点：

| `ConfigError` 变体 | 持有 | 由谁创建 |
|----------------------|-------|-----------|
| `FileRead(io::Error)` | 原始 I/O 错误 | `#[from]` 经 `?` 自动转换 |
| `Invalid { message }` | 人类可读说明 | 你的校验代码 |

现在可编写返回 `Result<T, ConfigError>` 的函数：

```rust
fn read_config(path: &str) -> Result<String, ConfigError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → ConfigError::FileRead
    if content.is_empty() {
        return Err(ConfigError::Invalid {
            message: "config file is empty".to_string(),
        });
    }
    Ok(content)
}
```

> **🟢 自学检查点：** 继续前请确认能回答：
> 1. 为何 `read_to_string` 上的 `?` 能工作？（因为 `#[from]` 生成 `impl From<io::Error> for ConfigError`）
> 2. 若添加第三变体 `MissingKey(String)`，需改哪些代码？（只需加变体；现有代码仍可编译）

## Crate 级错误类型与 Result 别名

项目超出单文件后，会将各 module 级错误合并为 **crate 级错误类型**。这是生产 Rust 的标准模式。下面从上面的 `ConfigError` 构建。

真实 Rust 项目中，每个 crate（或重要 module）定义自己的 `Error` 枚举与 `Result` 类型别名。这是惯用模式——类似 C++ 中为每库定义异常层次与 `using Result = std::expected<T, Error>`。

### 模式

```rust
// src/error.rs  (or at the top of lib.rs)
use thiserror::Error;

/// Every error this crate can produce.
#[derive(Error, Debug)]
pub enum Error {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),          // auto-converts via From

    #[error("JSON parse error: {0}")]
    Json(#[from] serde_json::Error),     // auto-converts via From

    #[error("Invalid sensor id: {0}")]
    InvalidSensor(u32),                  // domain-specific variant

    #[error("Timeout after {ms} ms")]
    Timeout { ms: u64 },
}

/// Crate-wide Result alias — saves typing throughout the crate.
pub type Result<T> = core::result::Result<T, Error>;
```

### 如何简化每个函数

无别名时需写：

```rust
// Verbose — error type repeated everywhere
fn read_sensor(id: u32) -> Result<f64, crate::Error> { ... }
fn parse_config(path: &str) -> Result<Config, crate::Error> { ... }
```

有别名后：

```rust
// Clean — just `Result<T>`
use crate::{Error, Result};

fn read_sensor(id: u32) -> Result<f64> {
    if id > 128 {
        return Err(Error::InvalidSensor(id));
    }
    let raw = std::fs::read_to_string(format!("/dev/sensor/{id}"))?; // io::Error → Error::Io
    let value: f64 = raw.trim().parse()
        .map_err(|_| Error::InvalidSensor(id))?;
    Ok(value)
}
```

`Io` 上的 `#[from]` 属性免费生成此 `impl`：

```rust
// Auto-generated by thiserror's #[from]
impl From<std::io::Error> for Error {
    fn from(source: std::io::Error) -> Self {
        Error::Io(source)
    }
}
```

这使 `?` 能工作：当某函数返回 `std::io::Error` 而你的函数返回 `Result<T>`（你的别名）时，编译器自动调用 `From::from()` 转换。

### 组合 module 级错误

较大 crate 按 module 拆分错误，在 crate 根组合：

```rust
// src/config/error.rs
#[derive(thiserror::Error, Debug)]
pub enum ConfigError {
    #[error("Missing key: {0}")]
    MissingKey(String),
    #[error("Invalid value for '{key}': {reason}")]
    InvalidValue { key: String, reason: String },
}

// src/error.rs  (crate-level)
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error(transparent)]               // delegates Display to inner error
    Config(#[from] crate::config::ConfigError),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}
pub type Result<T> = core::result::Result<T, Error>;
```

调用方仍可匹配具体配置错误：

```rust
match result {
    Err(Error::Config(ConfigError::MissingKey(k))) => eprintln!("Add '{k}' to config"),
    Err(e) => eprintln!("Other error: {e}"),
    Ok(v) => use_value(v),
}
```

### C++ 对照

| 概念 | C++ | Rust |
|---------|-----|------|
| 错误层次 | `class AppError : public std::runtime_error` | `#[derive(thiserror::Error)] enum Error { ... }` |
| 返回错误 | `std::expected<T, Error>` 或 `throw` | `fn foo() -> Result<T>` |
| 转换错误 | 手动 `try/catch` + 重抛 | `#[from]` + `?`——零样板 |
| Result 别名 | `template<class T> using Result = std::expected<T, Error>;` | `pub type Result<T> = core::result::Result<T, Error>;` |
| 错误消息 | 覆盖 `what()` | `#[error("...")]`——编译进 `Display` impl |

