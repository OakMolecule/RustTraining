## Rust 宏：从预处理器到元编程 {#rust-macros-from-preprocessor-to-metaprogramming}

> **你将学到：** Rust 宏如何工作、何时用宏而非函数或泛型，以及它们如何替代 C/C++ 预处理器。读完本章你能编写自己的 `macro_rules!` 宏，并理解 `#[derive(Debug)]` 底层做了什么。

宏是你在 Rust 中最早遇到的东西之一（第一行就是 `println!("hello")`），却是多数课程最后才讲的内容。本章补上这一课。

### 为何需要宏

函数与泛型处理 Rust 中大部分代码复用。宏填补类型系统够不着的地方：

| 需求 | 函数/泛型？ | 宏？ | 原因 |
|------|-------------------|--------|-----|
| 计算一个值 | ✅ `fn max<T: Ord>(a: T, b: T) -> T` | — | 类型系统能处理 |
| 接受可变数量参数 | ❌ Rust 无可变参数函数 | ✅ `println!("{} {}", a, b)` | 宏可接受任意数量 token |
| 生成重复的 `impl` 块 | ❌ 单靠泛型做不到 | ✅ `macro_rules!` | 宏在编译期生成代码 |
| 编译期运行代码 | ❌ `const fn` 有限制 | ✅ 过程宏 | 编译期可运行完整 Rust 代码 |
| 条件包含代码 | ❌ | ✅ `#[cfg(...)]` | 属性宏控制编译 |

若你来自 C/C++，可把宏视为*预处理器唯一正确的替代* — 区别在于它们操作语法树而非原始文本，因此是卫生的（不会意外名称冲突）且类型感知。

> **面向 C 开发者：** Rust 宏完全替代 `#define`。没有文本预处理器。完整预处理器 → Rust 映射见 [ch18](ch18-cpp-rust-semantic-deep-dives.md)。

---

## 用 `macro_rules!` 声明宏 {#declarative-macros-with-macro_rules}

声明宏（也称「按示例宏」）是 Rust 最常见的宏形式。它们对语法做模式匹配，类似对值做 `match`。

### 基本语法

```rust
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
}

fn main() {
    say_hello!();  // Expands to: println!("Hello!");
}
```

名称后的 `!` 告诉你（和编译器）这是宏调用。

### 带参数的模式匹配

宏用片段说明符对 *token 树* 做匹配：

```rust
macro_rules! greet {
    // Pattern 1: no arguments
    () => {
        println!("Hello, world!");
    };
    // Pattern 2: one expression argument
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

fn main() {
    greet!();           // "Hello, world!"
    greet!("Rust");     // "Hello, Rust!"
}
```

#### 片段说明符速查

| 说明符 | 匹配 | 示例 |
|-----------|---------|---------|
| `$x:expr` | 任意表达式 | `42`, `a + b`, `foo()` |
| `$x:ty` | 类型 | `i32`, `Vec<String>`, `&str` |
| `$x:ident` | 标识符 | `foo`, `my_var` |
| `$x:pat` | 模式 | `Some(x)`, `_`, `(a, b)` |
| `$x:stmt` | 语句 | `let x = 5;` |
| `$x:block` | 块 | `{ println!("hi"); 42 }` |
| `$x:literal` | 字面量 | `42`, `"hello"`, `true` |
| `$x:tt` | 单个 token 树 | 任意 — 通配符 |
| `$x:item` | 项（fn、struct、impl 等） | `fn foo() {}` |

### 重复 — 杀手级特性

C/C++ 宏不能循环。Rust 宏可以重复模式：

```rust
macro_rules! make_vec {
    // Match zero or more comma-separated expressions
    ( $( $element:expr ),* ) => {
        {
            let mut v = Vec::new();
            $( v.push($element); )*  // Repeat for each matched element
            v
        }
    };
}

fn main() {
    let v = make_vec![1, 2, 3, 4, 5];
    println!("{v:?}");  // [1, 2, 3, 4, 5]
}
```

`$( ... ),*` 语法表示「匹配零个或多个该模式，逗号分隔」。展开中的 `$( ... )*` 对每个匹配重复一次体。

> **标准库里的 `vec![]` 就是这样实现的。** 实际源码为：
> ```rust
> macro_rules! vec {
>     () => { Vec::new() };
>     ($elem:expr; $n:expr) => { vec::from_elem($elem, $n) };
>     ($($x:expr),+ $(,)?) => { <[_]>::into_vec(Box::new([$($x),+])) };
> }
> ```
> 末尾的 `$(,)?` 允许可选的尾逗号。

#### 重复运算符

| 运算符 | 含义 | 示例 |
|----------|---------|---------|
| `$( ... )*` | 零个或多个 | `vec![]`, `vec![1]`, `vec![1, 2, 3]` |
| `$( ... )+` | 一个或多个 | 至少需要一个元素 |
| `$( ... )?` | 零个或一个 | 可选元素 |

### 实用示例：`hashmap!` 构造器

标准库有 `vec![]` 但没有 `hashmap!{}`。我们来写一个：

```rust
macro_rules! hashmap {
    ( $( $key:expr => $value:expr ),* $(,)? ) => {
        {
            let mut map = std::collections::HashMap::new();
            $( map.insert($key, $value); )*
            map
        }
    };
}

fn main() {
    let scores = hashmap! {
        "Alice" => 95,
        "Bob" => 87,
        "Carol" => 92,  // trailing comma OK thanks to $(,)?
    };
    println!("{scores:?}");
}
```

### 实用示例：诊断检查宏

嵌入式/诊断代码中的常见模式 — 检查条件并返回错误：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum DiagError {
    #[error("Check failed: {0}")]
    CheckFailed(String),
}

macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_diagnostics(temp: f64, voltage: f64) -> Result<(), DiagError> {
    diag_check!(temp < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    diag_check!(voltage < 1.5, "Rail voltage too high");
    println!("All checks passed");
    Ok(())
}
```

> **C/C++ 对比：**
> ```c
> // C preprocessor — textual substitution, no type safety, no hygiene
> #define DIAG_CHECK(cond, msg) \
>     do { if (!(cond)) { log_error(msg); return -1; } } while(0)
> ```
> Rust 版本返回正确的 `Result` 类型，无双求值风险，编译器会检查 `$cond` 是否为 `bool` 表达式。

### 卫生性：为何 Rust 宏安全

C/C++ 宏 bug 常来自名称冲突：

```c
// C: dangerous — `x` could shadow the caller's `x`
#define SQUARE(x) ((x) * (x))
int x = 5;
int result = SQUARE(x++);  // UB: x incremented twice!
```

Rust 宏是**卫生的** — 宏内创建的变量不会泄漏到外面：

```rust
macro_rules! make_x {
    () => {
        let x = 42;  // This `x` is scoped to the macro expansion
    };
}

fn main() {
    let x = 10;
    make_x!();
    println!("{x}");  // Prints 10, not 42 — hygiene prevents collision
}
```

宏的 `x` 与调用者的 `x` 被编译器视为不同变量，尽管同名。**C 预处理器做不到这一点。**

---

## 常见标准库宏 {#common-standard-library-macros}

你从第 1 章就在用这些 — 它们实际做什么：

| 宏 | 作用 | 展开为（简化） |
|-------|-------------|------------------------|
| `println!("{}", x)` | 格式化打印到 stdout 并换行 | `std::io::_print(format_args!(...))` |
| `eprintln!("{}", x)` | 打印到 stderr 并换行 | 同上但到 stderr |
| `format!("{}", x)` | 格式化为 `String` | 分配并返回 `String` |
| `vec![1, 2, 3]` | 用元素创建 `Vec` | `Vec::from([1, 2, 3])`（近似） |
| `todo!()` | 标记未完成代码 | `panic!("not yet implemented")` |
| `unimplemented!()` | 标记故意未实现代码 | `panic!("not implemented")` |
| `unreachable!()` | 标记编译器无法证明不可达的代码 | `panic!("unreachable")` |
| `assert!(cond)` | 条件为假则 panic | `if !cond { panic!(...) }` |
| `assert_eq!(a, b)` | 值不等则 panic | 失败时显示两个值 |
| `dbg!(expr)` | 将表达式与值打印到 stderr 并返回值 | `eprintln!("[file:line] expr = {:#?}", &expr); expr` |
| `include_str!("file.txt")` | 编译期将文件内容嵌入为 `&str` | 编译时读取文件 |
| `include_bytes!("data.bin")` | 编译期将文件内容嵌入为 `&[u8]` | 编译时读取文件 |
| `cfg!(condition)` | 编译期条件作为 `bool` | 根据目标为 `true` 或 `false` |
| `env!("VAR")` | 编译期读取环境变量 | 未设置则编译失败 |
| `concat!("a", "b")` | 编译期拼接字面量 | `"ab"` |

### `dbg!` — 日常调试宏

```rust
fn factorial(n: u32) -> u32 {
    if dbg!(n <= 1) {     // Prints: [src/main.rs:2] n <= 1 = false
        dbg!(1)           // Prints: [src/main.rs:3] 1 = 1
    } else {
        dbg!(n * factorial(n - 1))  // Prints intermediate values
    }
}

fn main() {
    dbg!(factorial(4));   // Prints all recursive calls with file:line
}
```

`dbg!` 返回它包裹的值，因此可插入任意位置而不改变程序行为。它打印到 stderr（非 stdout），不干扰程序输出。**提交代码前删除所有 `dbg!` 调用。**

### 格式字符串语法

由于 `println!`、`format!`、`eprintln!` 和 `write!` 共用同一套格式机制，速查如下：

```rust
let name = "sensor";
let value = 3.14159;
let count = 42;

println!("{name}");                    // Variable by name (Rust 1.58+)
println!("{}", name);                  // Positional
println!("{value:.2}");                // 2 decimal places: "3.14"
println!("{count:>10}");               // Right-aligned, width 10: "        42"
println!("{count:0>10}");              // Zero-padded: "0000000042"
println!("{count:#06x}");              // Hex with prefix: "0x002a"
println!("{count:#010b}");             // Binary with prefix: "0b00101010"
println!("{value:?}");                 // Debug format
println!("{value:#?}");                // Pretty-printed Debug format
```

> **面向 C 开发者：** 可把它当作类型安全的 `printf` — 编译器检查 `{:.2}` 是否用于浮点数而非字符串。没有 `%s`/`%d` 格式不匹配 bug。
>
> **面向 C++ 开发者：** 这用一条可读格式字符串替代 `std::cout << std::fixed << std::setprecision(2) << value`。

---

## Derive 宏 {#derive-macros}

你在本书几乎每个结构体上都见过 `#[derive(...)]`：

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}
```

`#[derive(Debug)]` 是一种 **derive 宏** — 自动生成 trait 实现的过程宏。它生成的大致如下（简化）：

```rust
// What #[derive(Debug)] generates for Point:
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

没有 `#[derive(Debug)]`，你得为每个结构体手写该 `impl` 块。

### 常用 derive trait

| Derive | 生成内容 | 何时使用 |
|--------|-------------------|-------------|
| `Debug` | `{:?}` 格式化 | 几乎总是 — 便于调试打印 |
| `Clone` | `.clone()` 方法 | 需要复制值时 |
| `Copy` | 赋值时隐式复制 | 小的、仅栈上的类型（整数、`[f64; 3]`） |
| `PartialEq` / `Eq` | `==` 与 `!=` 运算符 | 需要相等比较时 |
| `PartialOrd` / `Ord` | `<`、`>`、`<=`、`>=` 运算符 | 需要排序时 |
| `Hash` | 用于 `HashMap`/`HashSet` 键的哈希 | 用作 map 键的类型 |
| `Default` | `Type::default()` 构造器 | 有合理零/空值的类型 |
| `serde::Serialize` / `Deserialize` | JSON/TOML 等序列化 | 跨 API 边界的数据类型 |

### derive 决策树

```text
Should I derive it?
  │
  ├── Does my type contain only types that implement the trait?
  │     ├── Yes → #[derive] will work
  │     └── No  → Write a manual impl (or skip it)
  │
  └── Will users of my type reasonably expect this behavior?
        ├── Yes → Derive it (Debug, Clone, PartialEq are almost always reasonable)
        └── No  → Don't derive (e.g., don't derive Copy for a type with a file handle)
```

> **C++ 对比：** `#[derive(Clone)]` 类似自动生成正确的拷贝构造函数。`#[derive(PartialEq)]` 类似自动生成逐字段比较的 `operator==` — C++20 的 `= default` 三路比较运算符终于提供了类似能力。

---

## 属性宏 {#attribute-macros}

属性宏变换所附的项。你已经用过几种：

```rust
#[test]                    // Marks a function as a test
fn test_addition() {
    assert_eq!(2 + 2, 4);
}

#[cfg(target_os = "linux")] // Conditionally includes this function
fn linux_only() { /* ... */ }

#[derive(Debug)]            // Generates Debug implementation
struct MyType { /* ... */ }

#[allow(dead_code)]         // Suppresses a compiler warning
fn unused_helper() { /* ... */ }

#[must_use]                 // Warn if return value is discarded
fn compute_checksum(data: &[u8]) -> u32 { /* ... */ }
```

常见内置属性：

| 属性 | 用途 |
|-----------|---------|
| `#[test]` | 标记为测试函数 |
| `#[cfg(...)]` | 条件编译 |
| `#[derive(...)]` | 自动生成 trait impl |
| `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` | 控制 lint 级别 |
| `#[must_use]` | 未使用返回值时警告 |
| `#[inline]` / `#[inline(always)]` | 提示内联函数 |
| `#[repr(C)]` | 使用 C 兼容内存布局（用于 FFI） |
| `#[no_mangle]` | 不修饰符号名（用于 FFI） |
| `#[deprecated]` | 标记为已弃用，可选消息 |

> **面向 C/C++ 开发者：** 属性替代了预处理器指令（`#pragma`、`__attribute__((...))`）和编译器扩展的混合体。它们是语言语法的一部分，不是后装扩展。

---

## 过程宏（概念概览）{#procedural-macros-conceptual-overview}

过程宏（「proc macro」）是作为*独立 Rust 程序*在编译期运行并生成代码的宏。比 `macro_rules!` 更强大也更复杂。

有三种：

| 种类 | 语法 | 示例 | 作用 |
|------|--------|---------|-------------|
| **函数式** | `my_macro!(...)` | `sql!(SELECT * FROM users)` | 解析自定义语法，生成 Rust 代码 |
| **Derive** | `#[derive(MyTrait)]` | `#[derive(Serialize)]` | 从结构体定义生成 trait impl |
| **属性** | `#[my_attr]` | `#[tokio::main]`、`#[instrument]` | 变换被注解的项 |

### 你已经用过过程宏

- `thiserror` 的 `#[derive(Error)]` — 为错误枚举生成 `Display` 与 `From` impl
- `serde` 的 `#[derive(Serialize, Deserialize)]` — 生成序列化代码
- `#[tokio::main]` — 将 `async fn main()` 变换为运行时设置 + block_on
- `#[test]` — 由测试框架注册（内置过程宏）

### 何时编写自己的过程宏

本课程中你可能不需要写过程宏。它们在以下情况有用：
- 需要在编译期检查结构体字段/枚举变体（derive 宏）
- 构建领域特定语言（函数式宏）
- 需要变换函数签名（属性宏）

对大多数代码，`macro_rules!` 或普通函数就够。

> **C++ 对比：** 过程宏承担 C++ 中代码生成器、模板元编程以及 `protoc` 等外部工具的角色。区别在于过程宏是 cargo 构建流水线的一部分 — 无外部构建步骤，无 CMake 自定义命令。

---

## 何时用何物：宏 vs 函数 vs 泛型 {#when-to-use-what-macros-vs-functions-vs-generics}

```text
Need to generate code?
  │
  ├── No → Use a function or generic function
  │         (simpler, better error messages, IDE support)
  │
  └── Yes ─┬── Variable number of arguments?
            │     └── Yes → macro_rules! (e.g., println!, vec!)
            │
            ├── Repetitive impl blocks for many types?
            │     └── Yes → macro_rules! with repetition
            │
            ├── Need to inspect struct fields?
            │     └── Yes → Derive macro (proc macro)
            │
            ├── Need custom syntax (DSL)?
            │     └── Yes → Function-like proc macro
            │
            └── Need to transform a function/struct?
                  └── Yes → Attribute proc macro
```

**一般准则：** 若函数或泛型能做到，就不要用宏。宏的错误信息更差、宏体内无 IDE 自动补全，也更难调试。

---

## 练习 {#exercises}

### 🟢 练习 1：`min!` 宏

编写 `min!` 宏：
- `min!(a, b)` 返回两个值中较小者
- `min!(a, b, c)` 返回三个值中最小者
- 适用于任何实现 `PartialOrd` 的类型

**提示：** 你的 `macro_rules!` 需要两个 match 分支。

<details><summary>解答（点击展开）</summary>

```rust
macro_rules! min {
    ($a:expr, $b:expr) => {
        if $a < $b { $a } else { $b }
    };
    ($a:expr, $b:expr, $c:expr) => {
        min!(min!($a, $b), $c)
    };
}

fn main() {
    println!("{}", min!(3, 7));        // 3
    println!("{}", min!(9, 2, 5));     // 2
    println!("{}", min!(1.5, 0.3));    // 0.3
}
```

**说明：** 生产代码优先用 `std::cmp::min` 或 `a.min(b)`。本练习演示多分支宏的机制。

</details>

### 🟡 练习 2：从零实现 `hashmap!`

不参考上文示例，编写 `hashmap!` 宏：
- 从 `key => value` 对创建 `HashMap`
- 支持尾逗号
- 适用于任何可哈希键类型

测试：
```rust
let m = hashmap! {
    "name" => "Alice",
    "role" => "Engineer",
};
assert_eq!(m["name"], "Alice");
assert_eq!(m.len(), 2);
```

<details><summary>解答（点击展开）</summary>

```rust
use std::collections::HashMap;

macro_rules! hashmap {
    ( $( $key:expr => $val:expr ),* $(,)? ) => {{
        let mut map = HashMap::new();
        $( map.insert($key, $val); )*
        map
    }};
}

fn main() {
    let m = hashmap! {
        "name" => "Alice",
        "role" => "Engineer",
    };
    assert_eq!(m["name"], "Alice");
    assert_eq!(m.len(), 2);
    println!("Tests passed!");
}
```

</details>

### 🟡 练习 3：浮点比较的 `assert_approx_eq!`

编写宏 `assert_approx_eq!(a, b, epsilon)`，当 `|a - b| > epsilon` 时 panic。用于精确相等会失败的浮点计算测试。

测试：
```rust
assert_approx_eq!(0.1 + 0.2, 0.3, 1e-10);        // Should pass
assert_approx_eq!(3.14159, std::f64::consts::PI, 1e-4); // Should pass
// assert_approx_eq!(1.0, 2.0, 0.5);              // Should panic
```

<details><summary>解答（点击展开）</summary>

```rust
macro_rules! assert_approx_eq {
    ($a:expr, $b:expr, $eps:expr) => {
        let (a, b, eps) = ($a as f64, $b as f64, $eps as f64);
        let diff = (a - b).abs();
        if diff > eps {
            panic!(
                "assertion failed: |{} - {}| = {} > {} (epsilon)",
                a, b, diff, eps
            );
        }
    };
}

fn main() {
    assert_approx_eq!(0.1 + 0.2, 0.3, 1e-10);
    assert_approx_eq!(3.14159, std::f64::consts::PI, 1e-4);
    println!("All float comparisons passed!");
}
```

</details>

### 🔴 练习 4：`impl_display_for_enum!`

编写宏，为简单 C 风格枚举生成 `Display` 实现。给定：

```rust
impl_display_for_enum! {
    enum Color {
        Red => "red",
        Green => "green",
        Blue => "blue",
    }
}
```

应同时生成 `enum Color { Red, Green, Blue }` 定义，以及将每个变体映射到字符串的 `impl Display for Color`。

**提示：** 需要 `$( ... ),*` 重复和多种片段说明符。

<details><summary>解答（点击展开）</summary>

```rust
use std::fmt;

macro_rules! impl_display_for_enum {
    (enum $name:ident { $( $variant:ident => $display:expr ),* $(,)? }) => {
        #[derive(Debug, Clone, Copy, PartialEq)]
        enum $name {
            $( $variant ),*
        }

        impl fmt::Display for $name {
            fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
                match self {
                    $( $name::$variant => write!(f, "{}", $display), )*
                }
            }
        }
    };
}

impl_display_for_enum! {
    enum Color {
        Red => "red",
        Green => "green",
        Blue => "blue",
    }
}

fn main() {
    let c = Color::Green;
    println!("Color: {c}");          // "Color: green"
    println!("Debug: {c:?}");        // "Debug: Green"
    assert_eq!(format!("{}", Color::Red), "red");
    println!("All tests passed!");
}
```

</details>
