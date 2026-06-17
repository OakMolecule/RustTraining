## C++ → Rust 语义深入 {#c-cpp-rust-semantic-deep-dives}

> **你将学到：** 对没有明显 Rust 等价物的 C++ 概念的详细映射 — 四种命名转换、SFINAE 与 trait 约束、CRTP 与关联类型，以及翻译过程中的其他常见摩擦点。

以下各节映射没有明显 1:1 Rust
等价物的 C++ 概念。这些差异常在 C++ 程序员做翻译工作时造成困扰。

### 转换层次：四种 C++ 转换 → Rust 等价物

C++ 有四种命名转换。Rust 用不同、更明确的机制替代它们：

```cpp
// C++ casting hierarchy
int i = static_cast<int>(3.14);            // 1. Numeric / up-cast
Derived* d = dynamic_cast<Derived*>(base); // 2. Runtime downcasting
int* p = const_cast<int*>(cp);              // 3. Cast away const
auto* raw = reinterpret_cast<char*>(&obj); // 4. Bit-level reinterpretation
```

| C++ 转换 | Rust 等价 | 安全性 | 说明 |
|----------|----------------|--------|-------|
| `static_cast`（数值） | `as` 关键字 | 安全但可能截断/环绕 | `let i = 3.14_f64 as i32;` — 截断为 3 |
| `static_cast`（数值，有检查） | `From`/`Into` | 安全，编译期验证 | `let i: i32 = 42_u8.into();` — 仅拓宽 |
| `static_cast`（数值，可失败） | `TryFrom`/`TryInto` | 安全，返回 `Result` | `let i: u8 = 300_u16.try_into()?;` — 返回 Err |
| `dynamic_cast`（向下转换） | 对枚举 `match` / `Any::downcast_ref` | 安全 | 枚举用模式匹配；trait 对象用 `Any` |
| `const_cast` | 无等价物 | | Rust 在 Safe Rust 中无法将 `&` 转为 `&mut`。内部可变性用 `Cell`/`RefCell` |
| `reinterpret_cast` | `std::mem::transmute` | **`unsafe`** | 重新解释位模式。几乎总是错的 — 优先 `from_le_bytes()` 等 |

```rust
// Rust equivalents:

// 1. Numeric casts — prefer From/Into over `as`
let widened: u32 = 42_u8.into();             // Infallible widening — always prefer
let truncated = 300_u16 as u8;                // ⚠ Wraps to 44! Silent data loss
let checked: Result<u8, _> = 300_u16.try_into(); // Err — safe fallible conversion

// 2. Downcast: enum (preferred) or Any (when needed for type erasure)
use std::any::Any;

fn handle_any(val: &dyn Any) {
    if let Some(s) = val.downcast_ref::<String>() {
        println!("Got string: {s}");
    } else if let Some(n) = val.downcast_ref::<i32>() {
        println!("Got int: {n}");
    }
}

// 3. "const_cast" → interior mutability (no unsafe needed)
use std::cell::Cell;
struct Sensor {
    read_count: Cell<u32>,  // Mutate through &self
}
impl Sensor {
    fn read(&self) -> f64 {
        self.read_count.set(self.read_count.get() + 1); // &self, not &mut self
        42.0
    }
}

// 4. reinterpret_cast → transmute (almost never needed)
// Prefer safe alternatives:
let bytes: [u8; 4] = 0x12345678_u32.to_ne_bytes();  // ✅ Safe
let val = u32::from_ne_bytes(bytes);                   // ✅ Safe
// unsafe { std::mem::transmute::<u32, [u8; 4]>(val) } // ❌ Avoid
```

> **准则**：惯用 Rust 中，`as` 应少见（拓宽用 `From`/`Into`，
> 收窄用 `TryFrom`/`TryInto`），`transmute` 应极少见，`const_cast` 无等价物，因为内部可变性
> 类型使其不必要。

---

### 预处理器 → `cfg`、特性标志与 `macro_rules!`

C++ 大量依赖预处理器做条件编译、常量与代码生成。Rust 用一等语言特性全部替代。

#### `#define` 常量 → `const` 或 `const fn`

```cpp
// C++
#define MAX_RETRIES 5
#define BUFFER_SIZE (1024 * 64)
#define SQUARE(x) ((x) * (x))  // Macro — textual substitution, no type safety
```

```rust
// Rust — type-safe, scoped, no textual substitution
const MAX_RETRIES: u32 = 5;
const BUFFER_SIZE: usize = 1024 * 64;
const fn square(x: u32) -> u32 { x * x }  // Evaluated at compile time

// Can be used in const contexts:
const AREA: u32 = square(12);  // Computed at compile time
static BUFFER: [u8; BUFFER_SIZE] = [0; BUFFER_SIZE];
```

#### `#ifdef` / `#if` → `#[cfg()]` 与 `cfg!()`

```cpp
// C++
#ifdef DEBUG
    log_verbose("Step 1 complete");
#endif

#if defined(LINUX) && !defined(ARM)
    use_x86_path();
#else
    use_generic_path();
#endif
```

```rust
// Rust — attribute-based conditional compilation
#[cfg(debug_assertions)]
fn log_verbose(msg: &str) { eprintln!("[VERBOSE] {msg}"); }

#[cfg(not(debug_assertions))]
fn log_verbose(_msg: &str) { /* compiled away in release */ }

// Combine conditions:
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn use_x86_path() { /* ... */ }

#[cfg(not(all(target_os = "linux", target_arch = "x86_64")))]
fn use_generic_path() { /* ... */ }

// Runtime check (condition is still compile-time, but usable in expressions):
if cfg!(target_os = "windows") {
    println!("Running on Windows");
}
```

#### `Cargo.toml` 中的特性标志

```toml
# Cargo.toml — replace #ifdef FEATURE_FOO
[features]
default = ["json"]
json = ["dep:serde_json"]       # Optional dependency
verbose-logging = []            # Flag with no extra dependency
gpu-support = ["dep:cuda-sys"]  # Optional GPU support
```

```rust
// Conditional code based on feature flags:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose-logging")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose-logging"))]
macro_rules! verbose {
    ($($arg:tt)*) => { }; // Compiles to nothing
}
```

#### `#define MACRO(x)` → `macro_rules!`

```cpp
// C++ — textual substitution, notoriously error-prone
#define DIAG_CHECK(cond, msg) \
    do { if (!(cond)) { log_error(msg); return false; } } while(0)
```

```rust
// Rust — hygienic, type-checked, operates on syntax tree
macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            log_error($msg);
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_test() -> Result<(), DiagError> {
    diag_check!(temperature < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    Ok(())
}
```

| C++ 预处理器 | Rust 等价 | 优势 |
|-----------------|----------------|-----------|
| `#define PI 3.14` | `const PI: f64 = 3.14;` | 有类型、有作用域、调试器可见 |
| `#define MAX(a,b) ((a)>(b)?(a):(b))` | `macro_rules!` 或泛型 `fn max<T: Ord>` | 无双求值 bug |
| `#ifdef DEBUG` | `#[cfg(debug_assertions)]` | 编译器检查，无拼写风险 |
| `#ifdef FEATURE_X` | `#[cfg(feature = "x")]` | Cargo 管理特性；感知依赖 |
| `#include "header.h"` | `mod module;` + `use module::Item;` | 无 include guard、无循环包含 |
| `#pragma once` | 不需要 | 每个 `.rs` 文件是一个模块 — 只包含一次 |

---

### 头文件与 `#include` → 模块与 `use`

在 C++ 中，编译模型围绕文本包含：

```cpp
// widget.h — every translation unit that uses Widget includes this
#pragma once
#include <string>
#include <vector>

class Widget {
public:
    Widget(std::string name);
    void activate();
private:
    std::string name_;
    std::vector<int> data_;
};
```

```cpp
// widget.cpp — separate definition
#include "widget.h"
Widget::Widget(std::string name) : name_(std::move(name)) {}
void Widget::activate() { /* ... */ }
```

在 Rust 中，**没有头文件、没有前向声明、没有 include guard**：

```rust
// src/widget.rs — declaration AND definition in one file
pub struct Widget {
    name: String,         // Private by default
    data: Vec<i32>,
}

impl Widget {
    pub fn new(name: String) -> Self {
        Widget { name, data: Vec::new() }
    }
    pub fn activate(&self) { /* ... */ }
}
```

```rust
// src/main.rs — import by module path
mod widget;  // Tells compiler to include src/widget.rs
use widget::Widget;

fn main() {
    let w = Widget::new("sensor".to_string());
    w.activate();
}
```

| C++ | Rust | 为何更好 |
|-----|------|-----------------|
| `#include "foo.h"` | 父模块中 `mod foo;` + `use foo::Item;` | 非文本包含，无 ODR 违反 |
| `#pragma once` / include guard | 不需要 | 每个 `.rs` 文件是一个模块 — 只编译一次 |
| 前向声明 | 不需要 | 编译器看到整个 crate；顺序无关 |
| `class Foo;`（不完整类型） | 不需要 | 无声明/定义分离 |
| 每个类 `.h` + `.cpp` | 单个 `.rs` 文件 | 无声明/定义不匹配 bug |
| `using namespace std;` | `use std::collections::HashMap;` | 始终显式 — 无全局命名空间污染 |
| 嵌套 `namespace a::b` | 嵌套 `mod a { mod b { } }` 或 `a/b.rs` | 文件系统镜像模块树 |

---

### `friend` 与访问控制 → 模块可见性

C++ 用 `friend` 授予特定类或函数访问私有成员的权限。
Rust 没有 `friend` 关键字 — 而是**按模块划分可见性**：

```cpp
// C++
class Engine {
    friend class Car;   // Car can access private members
    int rpm_;
    void set_rpm(int r) { rpm_ = r; }
public:
    int rpm() const { return rpm_; }
};
```

```rust
// Rust — items in the same module can access all fields, no `friend` needed
mod vehicle {
    pub struct Engine {
        rpm: u32,  // Private to the module (not to the struct!)
    }

    impl Engine {
        pub fn new() -> Self { Engine { rpm: 0 } }
        pub fn rpm(&self) -> u32 { self.rpm }
    }

    pub struct Car {
        engine: Engine,
    }

    impl Car {
        pub fn new() -> Self { Car { engine: Engine::new() } }
        pub fn accelerate(&mut self) {
            self.engine.rpm = 3000; // ✅ Same module — direct field access
        }
        pub fn rpm(&self) -> u32 {
            self.engine.rpm  // ✅ Same module — can read private field
        }
    }
}

fn main() {
    let mut car = vehicle::Car::new();
    car.accelerate();
    // car.engine.rpm = 9000;  // ❌ Compile error: `engine` is private
    println!("RPM: {}", car.rpm()); // ✅ Public method on Car
}
```

| C++ 访问 | Rust 等价 | 作用域 |
|-----------|----------------|-------|
| `private` | （默认，无关键字） | 仅同一模块内可访问 |
| `protected` | 无直接等价 | 用 `pub(super)` 供父模块访问 |
| `public` | `pub` | 到处可访问 |
| `friend class Foo` | 将 `Foo` 放在同一模块 | 模块级隐私替代 friend |
| — | `pub(crate)` | crate 内可见，外部依赖不可见 |
| — | `pub(super)` | 仅父模块可见 |
| — | `pub(in crate::path)` | 在指定模块子树内可见 |

> **要点**：C++ 隐私按类划分。Rust 隐私按模块划分。
> 这意味着通过选择哪些类型放在同一模块来控制访问 —
> 同处一模块的类型可完全访问彼此的私有字段。

---

### `volatile` → 原子操作与 `read_volatile`/`write_volatile`

在 C++ 中，`volatile` 告诉编译器不要优化掉读写 — 通常用于内存映射硬件寄存器。**Rust 没有 `volatile` 关键字。**

```cpp
// C++: volatile for hardware registers
volatile uint32_t* const GPIO_REG = reinterpret_cast<volatile uint32_t*>(0x4002'0000);
*GPIO_REG = 0x01;              // Write not optimized away
uint32_t val = *GPIO_REG;     // Read not optimized away
```

```rust
// Rust: explicit volatile operations — only in unsafe code
use std::ptr;

const GPIO_REG: *mut u32 = 0x4002_0000 as *mut u32;

// SAFETY: GPIO_REG is a valid memory-mapped I/O address.
unsafe {
    ptr::write_volatile(GPIO_REG, 0x01);   // Write not optimized away
    let val = ptr::read_volatile(GPIO_REG); // Read not optimized away
}
```

对于**并发共享状态**（C++ `volatile` 的另一常见误用），Rust 使用原子类型：

```cpp
// C++: volatile is NOT sufficient for thread safety (common mistake!)
volatile bool stop_flag = false;  // ❌ Data race — UB in C++11+

// Correct C++:
std::atomic<bool> stop_flag{false};
```

```rust
// Rust: atomics are the only way to share mutable state across threads
use std::sync::atomic::{AtomicBool, Ordering};

static STOP_FLAG: AtomicBool = AtomicBool::new(false);

// From another thread:
STOP_FLAG.store(true, Ordering::Release);

// Check:
if STOP_FLAG.load(Ordering::Acquire) {
    println!("Stopping");
}
```

| C++ 用法 | Rust 等价 | 说明 |
|-----------|----------------|-------|
| 硬件寄存器用 `volatile` | `ptr::read_volatile` / `ptr::write_volatile` | 需要 `unsafe` — 适用于 MMIO |
| 线程信号用 `volatile` | `AtomicBool` / `AtomicU32` 等 | C++ 用 `volatile` 做这也错！ |
| `std::atomic<T>` | `std::sync::atomic::AtomicT` | 语义相同，顺序相同 |
| `std::atomic<T>::load(memory_order_acquire)` | `AtomicT::load(Ordering::Acquire)` | 1:1 映射 |

---

### `static` 变量 → `static`、`const`、`LazyLock`、`OnceLock`

#### 基本 `static` 与 `const`

```cpp
// C++
const int MAX_RETRIES = 5;                    // Compile-time constant
static std::string CONFIG_PATH = "/etc/app";  // Static init — order undefined!
```

```rust
// Rust
const MAX_RETRIES: u32 = 5;                   // Compile-time constant, inlined
static CONFIG_PATH: &str = "/etc/app";         // 'static lifetime, fixed address
```

#### 静态初始化顺序灾难

C++ 有一个著名问题：不同翻译单元中的全局构造函数以**未指定顺序**
执行。Rust 完全避免这一点 — `static` 值必须是编译期常量（无构造函数）。

运行时初始化的全局变量用 `LazyLock`（Rust 1.80+）或 `OnceLock`：

```rust
use std::sync::LazyLock;

// Equivalent to C++ `static std::regex` — initialized on first access, thread-safe
static CONFIG_REGEX: LazyLock<regex::Regex> = LazyLock::new(|| {
    regex::Regex::new(r"^[a-z]+_diag$").expect("invalid regex")
});

fn is_valid_diag(name: &str) -> bool {
    CONFIG_REGEX.is_match(name)  // First call initializes; subsequent calls are fast
}
```

```rust
use std::sync::OnceLock;

// OnceLock: initialized once, can be set from runtime data
static DB_CONN: OnceLock<String> = OnceLock::new();

fn init_db(connection_string: &str) {
    DB_CONN.set(connection_string.to_string())
        .expect("DB_CONN already initialized");
}

fn get_db() -> &'static str {
    DB_CONN.get().expect("DB not initialized")
}
```

| C++ | Rust | 说明 |
|-----|------|-------|
| `const int X = 5;` | `const X: i32 = 5;` | 均为编译期。Rust 需要类型注解 |
| `constexpr int X = 5;` | `const X: i32 = 5;` | Rust `const` 始终是 constexpr |
| `static int count = 0;`（文件作用域） | `static COUNT: AtomicI32 = AtomicI32::new(0);` | 可变 static 需要 `unsafe` 或原子类型 |
| `static std::string s = "hi";` | `static S: &str = "hi";` 或 `LazyLock<String>` | 简单情况无运行时构造函数 |
| `static MyObj obj;`（复杂初始化） | `static OBJ: LazyLock<MyObj> = LazyLock::new(\|\| { ... });` | 线程安全、惰性、无初始化顺序问题 |
| `thread_local` | `thread_local! { static X: Cell<u32> = Cell::new(0); }` | 语义相同 |

---

### `constexpr` → `const fn`

C++ `constexpr` 标记函数与变量供编译期求值。Rust
用 `const fn` 与 `const` 达到同样目的：

```cpp
// C++
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(5);  // Computed at compile time → 120
```

```rust
// Rust
const fn factorial(n: u32) -> u32 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}
const VAL: u32 = factorial(5);  // Computed at compile time → 120

// Also works in array sizes and match patterns:
const LOOKUP: [u32; 5] = [factorial(1), factorial(2), factorial(3),
                           factorial(4), factorial(5)];
```

| C++ | Rust | 说明 |
|-----|------|-------|
| `constexpr int f()` | `const fn f() -> i32` | 同样意图 — 可编译期求值 |
| `constexpr` 变量 | `const` 变量 | Rust `const` 始终是编译期 |
| `consteval`（C++20） | 无等价物 | `const fn` 也可在运行时运行 |
| `if constexpr`（C++17） | 无等价物（用 `cfg!` 或泛型） | trait 特化可覆盖部分用例 |
| `constinit`（C++20） | 带 const 初始化的 `static` | Rust `static` 默认必须 const 初始化 |

> **`const fn` 的当前限制**（截至 Rust 1.82 已稳定）：
> - 无 trait 方法（const 上下文中不能对 `Vec` 调用 `.len()`）
> - 无堆分配（`Box::new`、`Vec::new` 不能 const）
> - ~~无浮点运算~~ — **Rust 1.82 已稳定**
> - 不能用 `for` 循环（用递归或带手动索引的 `while`）

---

### SFINAE 与 `enable_if` → Trait 约束与 `where` 子句

在 C++ 中，SFINAE（替换失败不是错误）是条件泛型编程背后的机制。它强大但出了名的难读。Rust
完全用 **trait 约束**替代：

```cpp
// C++: SFINAE-based conditional function (pre-C++20)
template<typename T,
         std::enable_if_t<std::is_integral_v<T>, int> = 0>
T double_it(T val) { return val * 2; }

template<typename T,
         std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
T double_it(T val) { return val * 2.0; }

// C++20 concepts — cleaner but still verbose:
template<std::integral T>
T double_it(T val) { return val * 2; }
```

```rust
// Rust: trait bounds — readable, composable, excellent error messages
use std::ops::Mul;

fn double_it<T: Mul<Output = T> + From<u8>>(val: T) -> T {
    val * T::from(2)
}

// Or with where clause for complex bounds:
fn process<T>(val: T) -> String
where
    T: std::fmt::Display + Clone + Send,
{
    format!("Processing: {}", val)
}

// Conditional behavior via separate impls (replaces SFINAE overloads):
trait Describable {
    fn describe(&self) -> String;
}

impl Describable for u32 {
    fn describe(&self) -> String { format!("integer: {self}") }
}

impl Describable for f64 {
    fn describe(&self) -> String { format!("float: {self:.2}") }
}
```

| C++ 模板元编程 | Rust 等价 | 可读性 |
|-----------------------------|----------------|-------------|
| `std::enable_if_t<cond>` | `where T: Trait` | 🟢 清晰自然语言 |
| `std::is_integral_v<T>` | 数值 trait 或具体类型的约束 | 🟢 无 `_v` / `_t` 后缀 |
| SFINAE 重载集 | 分离的 `impl Trait for ConcreteType` 块 | 🟢 每个 impl 独立 |
| `if constexpr (std::is_same_v<T, int>)` | 通过 trait impl 特化 | 🟢 编译期分发 |
| C++20 `concept` | `trait` | 🟢 意图几乎相同 |
| `requires` 子句 | `where` 子句 | 🟢 位置与语法相似 |
| 模板深处编译失败 | 调用点 trait 不匹配即失败 | 🟢 无 200 行错误级联 |

> **要点**：C++ concepts（C++20）最接近 Rust trait。
> 若熟悉 C++20 concepts，可把 Rust trait 视为自 1.0 起就是一等语言特性的 concepts，
> 有一致的实现模型（trait impl）而非鸭子类型。

---

### `std::function` → 函数指针、`impl Fn` 与 `Box<dyn Fn>`

C++ `std::function<R(Args...)>` 是类型擦除的可调用对象。Rust 有三种选择，
各有权衡：

```cpp
// C++: one-size-fits-all (heap-allocated, type-erased)
#include <functional>
std::function<int(int)> make_adder(int n) {
    return [n](int x) { return x + n; };
}
```

```rust
// Rust Option 1: fn pointer — simple, no captures, no allocation
fn add_one(x: i32) -> i32 { x + 1 }
let f: fn(i32) -> i32 = add_one;
println!("{}", f(5)); // 6

// Rust Option 2: impl Fn — monomorphized, zero overhead, can capture
fn apply(val: i32, f: impl Fn(i32) -> i32) -> i32 { f(val) }
let n = 10;
let result = apply(5, |x| x + n);  // Closure captures `n`

// Rust Option 3: Box<dyn Fn> — type-erased, heap-allocated (like std::function)
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}
let adder = make_adder(10);
println!("{}", adder(5));  // 15

// Storing heterogeneous callables (like vector<function<int(int)>>):
let callbacks: Vec<Box<dyn Fn(i32) -> i32>> = vec![
    Box::new(|x| x + 1),
    Box::new(|x| x * 2),
    Box::new(make_adder(100)),
];
for cb in &callbacks {
    println!("{}", cb(5));  // 6, 10, 105
}
```

| 何时使用 | C++ 等价 | Rust 选择 |
|------------|---------------|-------------|
| 顶层函数，无捕获 | 函数指针 | `fn(Args) -> Ret` |
| 泛型函数接受可调用对象 | 模板参数 | `impl Fn(Args) -> Ret`（静态分发） |
| 泛型中的 trait 约束 | `template<typename F>` | `F: Fn(Args) -> Ret` |
| 存储可调用对象，类型擦除 | `std::function<R(Args)>` | `Box<dyn Fn(Args) -> Ret>` |
| 修改状态的可回调 | 可变 lambda 的 `std::function` | `Box<dyn FnMut(Args) -> Ret>` |
| 一次性回调（被消费） | 移动的 `std::function` | `Box<dyn FnOnce(Args) -> Ret>` |

> **性能说明**：`impl Fn` 零开销（单态化，类似 C++ 模板）。
> `Box<dyn Fn>` 与 `std::function` 开销相同（vtable + 堆分配）。
> 除非需要存储异构可调用对象，否则优先 `impl Fn`。

---

### 容器映射：C++ STL → Rust `std::collections`

| C++ STL 容器 | Rust 等价 | 说明 |
|------------------|----------------|-------|
| `std::vector<T>` | `Vec<T>` | API 几乎相同。Rust 默认做边界检查 |
| `std::array<T, N>` | `[T; N]` | 栈上固定大小数组 |
| `std::deque<T>` | `std::collections::VecDeque<T>` | 环形缓冲区。两端 push/pop 高效 |
| `std::list<T>` | `std::collections::LinkedList<T>` | Rust 中很少用 — `Vec` 几乎总是更快 |
| `std::forward_list<T>` | 无等价物 | 用 `Vec` 或 `VecDeque` |
| `std::unordered_map<K, V>` | `std::collections::HashMap<K, V>` | 默认 `SipHash`（抗 DoS） |
| `std::map<K, V>` | `std::collections::BTreeMap<K, V>` | B 树；键有序；要求 `K: Ord` |
| `std::unordered_set<T>` | `std::collections::HashSet<T>` | 要求 `T: Hash + Eq` |
| `std::set<T>` | `std::collections::BTreeSet<T>` | 有序集合；要求 `T: Ord` |
| `std::priority_queue<T>` | `std::collections::BinaryHeap<T>` | 默认最大堆（与 C++ 相同） |
| `std::stack<T>` | 带 `.push()` / `.pop()` 的 `Vec<T>` | 无需单独 stack 类型 |
| `std::queue<T>` | 带 `.push_back()` / `.pop_front()` 的 `VecDeque<T>` | 无需单独 queue 类型 |
| `std::string` | `String` | 保证 UTF-8，非 null 终止 |
| `std::string_view` | `&str` | 借用的 UTF-8 切片 |
| `std::span<T>`（C++20） | `&[T]` / `&mut [T]` | Rust 切片自 1.0 即为一等类型 |
| `std::tuple<A, B, C>` | `(A, B, C)` | 一等语法，可解构 |
| `std::pair<A, B>` | `(A, B)` | 就是二元组 |
| `std::bitset<N>` | 标准库无等价 | 用 `bitvec` crate 或 `[u8; N/8]` |

**主要差异**：
- Rust 的 `HashMap`/`HashSet` 要求 `K: Hash + Eq` — 编译器在类型层面强制，不像 C++ 用不可哈希键会在 STL 深处报模板错误
- `Vec` 索引（`v[i]`）默认越界会 panic。用 `.get(i)` 得到 `Option<&T>`，或迭代器完全避免边界检查
- 无 `std::multimap` 或 `std::multiset` — 用 `HashMap<K, Vec<V>>` 或 `BTreeMap<K, Vec<V>>`

---

### 异常安全 → Panic 安全

C++ 定义三级异常安全（Abrahams 保证）：

| C++ 级别 | 含义 | Rust 等价 |
|----------|---------|----------------|
| **No-throw** | 函数从不抛出 | 函数从不 panic（返回 `Result`） |
| **Strong**（提交或回滚） | 若抛出，状态不变 | 所有权模型使这很自然 — `?` 早退时，部分构建的值会被 drop |
| **Basic** | 若抛出，不变量保持 | Rust 默认 — `Drop` 运行，无泄漏 |

#### Rust 所有权模型如何帮助

```rust
// Strong guarantee for free — if file.write() fails, config is unchanged
fn update_config(config: &mut Config, path: &str) -> Result<(), Error> {
    let new_data = fetch_from_network()?; // Err → early return, config untouched
    let validated = validate(new_data)?;   // Err → early return, config untouched
    *config = validated;                   // Only reached on success (commit)
    Ok(())
}
```

在 C++ 中，强保证需要手动回滚或 copy-and-swap
惯用法。在 Rust 中，`?` 传播对大多数代码默认给出强保证。

#### `catch_unwind` — Rust 的 `catch(...)` 等价物

```rust
use std::panic;

// Catch a panic (like catch(...) in C++) — rarely needed
let result = panic::catch_unwind(|| {
    // Code that might panic
    let v = vec![1, 2, 3];
    v[10]  // Panics! (index out of bounds)
});

match result {
    Ok(val) => println!("Got: {val}"),
    Err(_) => eprintln!("Caught a panic — cleaned up"),
}
```

#### `UnwindSafe` — 标记 panic 安全类型

```rust
use std::panic::UnwindSafe;

// Types behind &mut are NOT UnwindSafe by default — the panic may have
// left them in a partially-modified state
fn safe_execute<F: FnOnce() + UnwindSafe>(f: F) {
    let _ = std::panic::catch_unwind(f);
}

// Use AssertUnwindSafe to override when you've audited the code:
use std::panic::AssertUnwindSafe;
let mut data = vec![1, 2, 3];
let _ = std::panic::catch_unwind(AssertUnwindSafe(|| {
    data.push(4);
}));
```

| C++ 异常模式 | Rust 等价 |
|-----------------------|-----------------|
| `throw MyException()` | `return Err(MyError::...)`（首选）或 `panic!("...")` |
| `try { } catch (const E& e)` | `match result { Ok(v) => ..., Err(e) => ... }` 或 `?` |
| `catch (...)` | `std::panic::catch_unwind(...)` |
| `noexcept` | `-> Result<T, E>`（错误是值，不是异常） |
| 栈展开中的 RAII 清理 | panic 展开时运行 `Drop::drop()` |
| `std::uncaught_exceptions()` | `std::thread::panicking()` |
| `-fno-exceptions` 编译选项 | `Cargo.toml` [profile] 中 `panic = "abort"` |

> **结论**：Rust 中大多数代码用 `Result<T, E>` 而非异常，
> 使错误路径显式且可组合。`panic!` 留给 bug
>（如 `assert!` 失败），不是常规错误。这意味着「异常安全」
> 基本上不是问题 — 所有权系统自动处理清理。

---

## C++ 到 Rust 迁移模式 {#c-to-rust-migration-patterns}

### 速查：C++ → Rust 惯用法映射

| **C++ 模式** | **Rust 惯用法** | **说明** |
|----------------|---------------|----------|
| `class Derived : public Base` | `enum Variant { A {...}, B {...} }` | 封闭集合优先用枚举 |
| `virtual void method() = 0` | `trait MyTrait { fn method(&self); }` | 用于开放/可扩展接口 |
| `dynamic_cast<Derived*>(ptr)` | `match value { Variant::A(data) => ..., }` | 穷尽，无运行时失败 |
| `vector<unique_ptr<Base>>` | `Vec<Box<dyn Trait>>` | 仅当真正需要多态时 |
| `shared_ptr<T>` | `Rc<T>` 或 `Arc<T>` | 优先 `Box<T>` 或拥有值 |
| `enable_shared_from_this<T>` | 竞技场模式（`Vec<T>` + 索引） | 彻底消除引用环 |
| 每个类中的 `Base* m_pFramework` | `fn execute(&mut self, ctx: &mut Context)` | 传上下文，不要存指针 |
| `try { } catch (...) { }` | `match result { Ok(v) => ..., Err(e) => ... }` | 或用 `?` 传播 |
| `std::optional<T>` | `Option<T>` | 必须 `match`，不能忘记 `None` |
| `const std::string&` 参数 | `&str` 参数 | 同时接受 `String` 与 `&str` |
| `enum class Foo { A, B, C }` | `enum Foo { A, B, C }` | Rust 枚举也可携带数据 |
| `auto x = std::move(obj)` | `let x = obj;` | 移动是默认，无需 `std::move` |
| CMake + make + lint | `cargo build / test / clippy / fmt` | 一个工具搞定一切 |

### 迁移策略
1. **从数据类型开始**：先翻译结构体与枚举 — 迫使你思考所有权
2. **工厂转枚举**：若工厂创建不同派生类型，多半应是 `enum` + `match`
3. **上帝对象拆成组合结构体**：将相关字段分组到专注的结构体
4. **指针换借用**：将存储的 `Base*` 转为带生命周期的 `&'a T` 借用
5. **谨慎使用 `Box<dyn Trait>`**：仅用于插件系统与测试 mock
6. **让编译器引导你**：Rust 错误信息很好 — 仔细阅读





