# Rust 最佳实践摘要 {#rust-best-practices-summary}

> **你将学到：** 编写地道 Rust 的实用指南 — 代码组织、命名约定、错误处理模式与文档。你会经常回顾的快速参考章节。

## 代码组织 {#code-organization}
- **优先小函数**：易于测试与推理
- **使用描述性名称**：`calculate_total_price()` 而非 `calc()`
- **将相关功能分组**：使用模块与独立文件
- **编写文档**：对公共 API 使用 `///`

## 错误处理 {#error-handling}
- **除非确定不会 panic，避免 `unwrap()`**：仅在 100% 确信时使用
```rust
// Bad: Can panic
let value = some_option.unwrap();

// Good: Handle the None case
let value = some_option.unwrap_or(default_value);
let value = some_option.unwrap_or_else(|| expensive_computation());
let value = some_option.unwrap_or_default(); // Uses Default trait

// For Result<T, E>
let value = some_result.unwrap_or(fallback_value);
let value = some_result.unwrap_or_else(|err| {
    eprintln!("Error occurred: {err}");
    default_value
});
```
- **用描述性消息使用 `expect()`**：当 unwrap 合理时，说明原因
```rust
let config = std::env::var("CONFIG_PATH")
    .expect("CONFIG_PATH environment variable must be set");
```
- **可失败操作返回 `Result<T, E>`**：让调用方决定如何处理错误
- **自定义错误类型使用 `thiserror`**：比手写实现更顺手
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Parse error: {message}")]
    Parse { message: String },
    
    #[error("Value {value} is out of range")]
    OutOfRange { value: i32 },
}
```
- **用 `?` 运算符链式传播错误**：沿调用栈向上传递
- **优先 `thiserror` 而非 `anyhow`**：我们团队约定用 `#[derive(thiserror::Error)]` 定义显式错误枚举，以便调用方匹配具体变体。`anyhow::Error` 适合快速原型，但会抹平错误类型，使调用方难以处理特定失败。库与生产代码用 `thiserror`；仅在只需打印错误的临时脚本或顶层二进制中用 `anyhow`。
- **何时 `unwrap()` 可接受**：
  - **单元测试**：`assert_eq!(result.unwrap(), expected)`
  - **原型**：会被替换的临时代码
  - **不会失败的操作**：能证明不会失败时
```rust
let numbers = vec![1, 2, 3];
let first = numbers.get(0).unwrap(); // Safe: we just created the vec with elements

// Better: Use expect() with explanation
let first = numbers.get(0).expect("numbers vec is non-empty by construction");
```
- **快速失败**：尽早检查前置条件并立即返回错误

## 内存管理 {#memory-management}
- **优先借用而非 clone**：尽可能用 `&T` 代替 clone
- **谨慎使用 `Rc<T>`**：仅在需要共享所有权时
- **限制生命周期**：用作用域 `{}` 控制值何时 drop
- **公共 API 避免 `RefCell<T>`**：内部可变性保持内部

## 性能 {#performance}
- **先 profile 再优化**：使用 `cargo bench` 与 profiling 工具
- **优先迭代器而非循环**：更可读且往往更快
- **需要所有权时用 `&str` 而非 `String`**
- **大栈对象考虑 `Box<T>`**：必要时移到堆上

## 应实现的基本 Trait {#essential-traits-to-implement}

### 每种类型都应考虑的核心 Trait {#core-trait-every-type-should-consider}

创建自定义类型时，考虑实现这些基础 Trait，使类型在 Rust 中感觉原生：

#### **Debug 与 Display** {#debug-and-display}
```rust
use std::fmt;

#[derive(Debug)]  // Automatic implementation for debugging
struct Person {
    name: String,
    age: u32,
}

// Manual Display implementation for user-facing output
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

// Usage:
let person = Person { name: "Alice".to_string(), age: 30 };
println!("{:?}", person);  // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);    // Display: Alice (age 30)
```

#### **Clone 与 Copy** {#clone-and-copy}
```rust
// Copy: Implicit duplication for small, simple types
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

// Clone: Explicit duplication for complex types
#[derive(Debug, Clone)]
struct Person {
    name: String,  // String doesn't implement Copy
    age: u32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Copy (implicit)

let person1 = Person { name: "Bob".to_string(), age: 25 };
let person2 = person1.clone();  // Clone (explicit)
```

#### **PartialEq 与 Eq** {#partialeq-and-eq}
```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, PartialEq)]
struct Temperature {
    celsius: f64,  // f64 doesn't implement Eq (due to NaN)
}

let id1 = UserId(123);
let id2 = UserId(123);
assert_eq!(id1, id2);  // Works because of PartialEq

let temp1 = Temperature { celsius: 20.0 };
let temp2 = Temperature { celsius: 20.0 };
assert_eq!(temp1, temp2);  // Works with PartialEq
```

#### **PartialOrd 与 Ord** {#partialord-and-ord}
```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

let high = Priority(1);
let low = Priority(10);
assert!(high < low);  // Lower numbers = higher priority

// Use in collections
let mut priorities = vec![Priority(5), Priority(1), Priority(8)];
priorities.sort();  // Works because Priority implements Ord
```

#### **Default** {#default}
```rust
#[derive(Debug, Default)]
struct Config {
    debug: bool,           // false (default)
    max_connections: u32,  // 0 (default)
    timeout: Option<u64>,  // None (default)
}

// Custom Default implementation
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            max_connections: 100,  // Custom default
            timeout: Some(30),     // Custom default
        }
    }
}

let config = Config::default();
let config = Config { debug: true, ..Default::default() };  // Partial override
```

#### **From 与 Into** {#from-and-into}
```rust
struct UserId(u64);
struct UserName(String);

// Implement From, and Into comes for free
impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

impl From<String> for UserName {
    fn from(name: String) -> Self {
        UserName(name)
    }
}

impl From<&str> for UserName {
    fn from(name: &str) -> Self {
        UserName(name.to_string())
    }
}

// Usage:
let user_id: UserId = 123u64.into();         // Using Into
let user_id = UserId::from(123u64);          // Using From
let username = UserName::from("alice");      // &str -> UserName
let username: UserName = "bob".into();       // Using Into
```

#### **TryFrom 与 TryInto** {#tryfrom-and-tryinto}
```rust
use std::convert::TryFrom;

struct PositiveNumber(u32);

#[derive(Debug)]
struct NegativeNumberError;

impl TryFrom<i32> for PositiveNumber {
    type Error = NegativeNumberError;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 0 {
            Ok(PositiveNumber(value as u32))
        } else {
            Err(NegativeNumberError)
        }
    }
}

// Usage:
let positive = PositiveNumber::try_from(42)?;     // Ok(PositiveNumber(42))
let error = PositiveNumber::try_from(-5);         // Err(NegativeNumberError)
```

#### **Serde（序列化）** {#serde-for-serialization}
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// Automatic JSON serialization/deserialization
let user = User {
    id: 1,
    name: "Alice".to_string(),
    email: "alice@example.com".to_string(),
};

let json = serde_json::to_string(&user)?;
let deserialized: User = serde_json::from_str(&json)?;
```

### Trait 实现清单 {#trait-implementation-checklist}

对新类型，可参考此清单：

```rust
#[derive(
    Debug,          // [OK] Always implement for debugging
    Clone,          // [OK] If the type should be duplicatable
    PartialEq,      // [OK] If the type should be comparable
    Eq,             // [OK] If comparison is reflexive/transitive
    PartialOrd,     // [OK] If the type has ordering
    Ord,            // [OK] If ordering is total
    Hash,           // [OK] If type will be used as HashMap key
    Default,        // [OK] If there's a sensible default value
)]
struct MyType {
    // fields...
}

// Manual implementations to consider:
impl Display for MyType { /* user-facing representation */ }
impl From<OtherType> for MyType { /* convenient conversion */ }
impl TryFrom<FallibleType> for MyType { /* fallible conversion */ }
```

### 何时不应实现 Trait {#when-not-to-implement-trait}

- **堆数据类型不要实现 Copy**：`String`、`Vec`、`HashMap` 等
- **值可能为 NaN 时不要实现 Eq**：含 `f32`/`f64` 的类型
- **没有合理默认值时不要实现 Default**：文件句柄、网络连接
- **clone 昂贵时不要实现 Clone**：大型数据结构（考虑 `Rc<T>`）

### 摘要：Trait 收益 {#summary-trait-benefits}

| Trait | 收益 | 何时使用 |
|-------|---------|-------------|
| `Debug` | `println!("{:?}", value)` | 几乎总是（极少数例外） |
| `Display` | `println!("{}", value)` | 面向用户的类型 |
| `Clone` | `value.clone()` | 显式复制有意义时 |
| `Copy` | 隐式复制 | 小型简单类型 |
| `PartialEq` | `==` 与 `!=` | 大多数类型 |
| `Eq` | 自反相等 | 数学上相等可靠时 |
| `PartialOrd` | `<`、`>`、`<=`、`>=` | 有自然序的类型 |
| `Ord` | `.sort()`、`BinaryHeap` | 全序时 |
| `Hash` | `HashMap` 键 | 用作 map 键的类型 |
| `Default` | `Default::default()` | 有明显默认值的类型 |
| `From/Into` | 便捷转换 | 常见类型转换 |
| `TryFrom/TryInto` | 可失败转换 | 可能失败的转换 |

----

----

