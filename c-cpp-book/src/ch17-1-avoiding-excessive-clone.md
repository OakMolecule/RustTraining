## 避免过度 `clone()` {#avoiding-excessive-clone}

> **你将学到：** 为何 `.clone()` 在 Rust 中是代码异味、如何通过重构所有权消除不必要拷贝，以及标志所有权设计问题的具体模式。

- 来自 C++ 时，`.clone()` 感觉像安全默认 — 「复制一下就行」。但过度 clone 掩盖所有权问题并损害性能。
- **经验法则**：若 clone 是为了满足借用检查器，多半需要重构所有权而非复制。

### 何时 `clone()` 是错误的 {#when-clone-is-wrong}

```rust
// BAD: Cloning a String just to pass it to a function that only reads it
fn log_message(msg: String) {  // Takes ownership unnecessarily
    println!("[LOG] {}", msg);
}
let message = String::from("GPU test passed");
log_message(message.clone());  // Wasteful: allocates a whole new String
log_message(message);           // Original consumed — clone was pointless
```

```rust
// GOOD: Accept a borrow — zero allocation
fn log_message(msg: &str) {    // Borrows, doesn't own
    println!("[LOG] {}", msg);
}
let message = String::from("GPU test passed");
log_message(&message);          // No clone, no allocation
log_message(&message);          // Can call again — message not consumed
```

### 真实示例：返回 `&str` 而非 clone {#real-example-returning-str-instead-of-cloning}
```rust
// Example: healthcheck.rs — returns a borrowed view, zero allocation
pub fn serial_or_unknown(&self) -> &str {
    self.serial.as_deref().unwrap_or(UNKNOWN_VALUE)
}

pub fn model_or_unknown(&self) -> &str {
    self.model.as_deref().unwrap_or(UNKNOWN_VALUE)
}
```
C++ 等价物会返回 `const std::string&` 或 `std::string_view` — 但 C++ 中两者都没有生命周期检查。Rust 中借用检查器保证返回的 `&str` 不会比 `self` 活得更久。

### 真实示例：静态字符串切片 — 完全不用堆 {#real-example-static-string-slices--no-heap-at-all}
```rust
// Example: healthcheck.rs — compile-time string tables
const HBM_SCREEN_RECIPES: &[&str] = &[
    "hbm_ds_ntd", "hbm_ds_ntd_gfx", "hbm_dt_ntd", "hbm_dt_ntd_gfx",
    "hbm_burnin_8h", "hbm_burnin_24h",
];
```
C++ 中通常是 `std::vector<std::string>`（首次使用时堆分配）。Rust 的 `&'static [&'static str]` 存在于只读内存 — 零运行时成本。

### 何时 `clone()` 是合适的 {#when-clone-is-appropriate}

| **情况** | **为何 clone 可接受** | **示例** |
|--------------|--------------------|-----------|
| 线程间 `Arc::clone()` | 仅增加引用计数（约 1 ns），不复制数据 | `let flag = stop_flag.clone();` |
| 将数据移入 spawn 的线程 | 线程需要自己的副本 | `let ctx = ctx.clone(); thread::spawn(move \|\| { ... })` |
| 从 `&self` 字段取出 | 不能从借用中移出 | 返回 owned `String` 时 `self.name.clone()` |
| 包在 `Option` 中的小 `Copy` 类型 | `.copied()` 比 `.clone()` 更清晰 | `Option<&u32>` → `Option<u32>` 用 `opt.get(0).copied()` |

### 真实示例：线程共享的 `Arc::clone` {#real-example-arcclone-for-thread-sharing}
```rust
// Example: workload.rs — Arc::clone is cheap (ref count bump)
let stop_flag = Arc::new(AtomicBool::new(false));
let stop_flag_clone = stop_flag.clone();   // ~1 ns, no data copied
let ctx_clone = ctx.clone();               // Clone context for move into thread

let sensor_handle = thread::spawn(move || {
    // ...uses stop_flag_clone and ctx_clone
});
```

### 清单：我该 clone 吗？ {#checklist-should-i-clone}
1. **能否接受 `&str` / `&T` 而非 `String` / `T`？** → 借用，不要 clone
2. **能否重构以避免两个所有者？** → 传引用或用作用域
3. **这是 `Arc::clone()` 吗？** → 可以，O(1)
4. **要把数据移入线程/闭包？** → clone 必要
5. **在热循环里 clone？** → profile，考虑借用或 `Cow<T>`

----

## `Cow<'a, T>`：写时克隆 — 能借则借，必须时才 clone {#cow-a-t-borrow-when-you-can-clone-when-you-must}

`Cow`（Clone on Write）是枚举，持有**借用引用**或**owned 值**。相当于「尽量免分配，修改时才分配」。C++ 没有直接等价物 — 最接近的是有时返回 `const std::string&`、有时返回 `std::string` 的函数。

### 为何需要 `Cow` {#why-cow-exists}

```rust
// Without Cow — you must choose: always borrow OR always clone
fn normalize(s: &str) -> String {          // Always allocates!
    if s.contains(' ') {
        s.replace(' ', "_")               // New String (allocation needed)
    } else {
        s.to_string()                     // Unnecessary allocation!
    }
}

// With Cow — borrow when unchanged, allocate only when modified
use std::borrow::Cow;

fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(' ') {
        Cow::Owned(s.replace(' ', "_"))    // Allocates (must modify)
    } else {
        Cow::Borrowed(s)                   // Zero allocation (passthrough)
    }
}
```

### `Cow` 如何工作 {#how-cow-works}

```rust
use std::borrow::Cow;

// Cow<'a, str> is essentially:
// enum Cow<'a, str> {
//     Borrowed(&'a str),     // Zero-cost reference
//     Owned(String),          // Heap-allocated owned value
// }

fn greet(name: &str) -> Cow<'_, str> {
    if name.is_empty() {
        Cow::Borrowed("stranger")         // Static string — no allocation
    } else if name.starts_with(' ') {
        Cow::Owned(name.trim().to_string()) // Modified — allocation needed
    } else {
        Cow::Borrowed(name)               // Passthrough — no allocation
    }
}

fn main() {
    let g1 = greet("Alice");     // Cow::Borrowed("Alice")
    let g2 = greet("");          // Cow::Borrowed("stranger")
    let g3 = greet(" Bob ");     // Cow::Owned("Bob")
    
    // Cow<str> implements Deref<Target = str>, so you can use it as &str:
    println!("Hello, {g1}!");    // Works — Cow auto-derefs to &str
    println!("Hello, {g2}!");
    println!("Hello, {g3}!");
}
```

### 真实用例：配置值规范化 {#real-world-use-case-config-value-normalization}

```rust
use std::borrow::Cow;

/// Normalize a SKU name: trim whitespace, lowercase.
/// Returns Cow::Borrowed if already normalized (zero allocation).
fn normalize_sku(sku: &str) -> Cow<'_, str> {
    let trimmed = sku.trim();
    if trimmed == sku && sku.chars().all(|c| c.is_lowercase() || !c.is_alphabetic()) {
        Cow::Borrowed(sku)   // Already normalized — no allocation
    } else {
        Cow::Owned(trimmed.to_lowercase())  // Needs modification — allocate
    }
}

fn main() {
    let s1 = normalize_sku("server-x1");   // Borrowed — zero alloc
    let s2 = normalize_sku("  Server-X1 "); // Owned — must allocate
    println!("{s1}, {s2}"); // "server-x1, server-x1"
}
```

### 何时使用 `Cow` {#when-to-use-cow}

| **情况** | **用 `Cow`？** |
|--------------|---------------|
| 函数大多原样返回输入 | ✅ 是 — 避免不必要 clone |
| 解析/规范化字符串（trim、小写、替换） | ✅ 是 — 输入常已合法 |
| 总是修改 — 每条路径都分配 | ❌ 否 — 直接返回 `String` |
| 简单透传（从不修改） | ❌ 否 — 直接返回 `&str` |
| 长期存在结构体中的数据 | ❌ 否 — 用 `String`（owned） |

> **C++ 对比**：`Cow<str>` 类似返回 `std::variant<std::string_view, std::string>` 的函数 — 但有自动 deref，访问值无样板代码。

----

## `Weak<T>`：打破引用循环 — Rust 的 `weak_ptr` {#weakt-breaking-reference-cycles--rusts-weak_ptr}

`Weak<T>` 是 C++ `std::weak_ptr<T>` 的 Rust 等价物。它持有对 `Rc<T>` 或 `Arc<T>` 的非 owning 引用。值可在仍有 `Weak` 引用时被释放 — 调用 `upgrade()` 若值已消失则返回 `None`。

### 为何需要 `Weak` {#why-weak-exists}

`Rc<T>` 与 `Arc<T>` 若两个值相互指向，会形成引用循环 — 两者引用计数永不为 0，都不会被 drop（内存泄漏）。`Weak` 打破循环：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: String,
    parent: RefCell<Weak<Node>>,      // Weak — doesn't prevent parent from dropping
    children: RefCell<Vec<Rc<Node>>>,  // Strong — parent owns children
}

impl Node {
    fn new(value: &str) -> Rc<Node> {
        Rc::new(Node {
            value: value.to_string(),
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(Vec::new()),
        })
    }

    fn add_child(parent: &Rc<Node>, child: &Rc<Node>) {
        // Child gets a weak reference to parent (no cycle)
        *child.parent.borrow_mut() = Rc::downgrade(parent);
        // Parent gets a strong reference to child
        parent.children.borrow_mut().push(Rc::clone(child));
    }
}

fn main() {
    let root = Node::new("root");
    let child = Node::new("child");
    Node::add_child(&root, &child);

    // Access parent from child via upgrade()
    if let Some(parent) = child.parent.borrow().upgrade() {
        println!("Child's parent: {}", parent.value); // "root"
    }
    
    println!("Root strong count: {}", Rc::strong_count(&root));  // 1
    println!("Root weak count: {}", Rc::weak_count(&root));      // 1
}
```

### C++ 对比 {#c-comparison}

```cpp
// C++ — weak_ptr to break shared_ptr cycle
struct Node {
    std::string value;
    std::weak_ptr<Node> parent;                  // Weak — no ownership
    std::vector<std::shared_ptr<Node>> children;  // Strong — owns children

    static auto create(const std::string& v) {
        return std::make_shared<Node>(Node{v, {}, {}});
    }
};

auto root = Node::create("root");
auto child = Node::create("child");
child->parent = root;          // weak_ptr assignment
root->children.push_back(child);

if (auto p = child->parent.lock()) {   // lock() → shared_ptr or null
    std::cout << "Parent: " << p->value << std::endl;
}
```

| C++ | Rust | 说明 |
|-----|------|-------|
| `shared_ptr<T>` | `Rc<T>`（单线程）/ `Arc<T>`（多线程） | 语义相同 |
| `weak_ptr<T>` | `Weak<T>` from `Rc::downgrade()` / `Arc::downgrade()` | 语义相同 |
| `weak_ptr::lock()` → `shared_ptr` 或 null | `Weak::upgrade()` → `Option<Rc<T>>` | 已 drop 则为 `None` |
| `shared_ptr::use_count()` | `Rc::strong_count()` | 含义相同 |

### 何时使用 `Weak` {#when-to-use-weak}

| **情况** | **模式** |
|--------------|-----------|
| 父子树关系 | 父持 `Rc<Child>`，子持 `Weak<Parent>` |
| 观察者模式 / 事件监听器 | 事件源持 `Weak<Observer>`，观察者持 `Rc<Source>` |
| 不阻止释放的缓存 | `HashMap<Key, Weak<Value>>` — 条目自然过期 |
| 图结构中的打破循环 | 交叉链接用 `Weak`，树边用 `Rc`/`Arc` |

> **新代码中优先 arena 模式**（案例研究 2）而非 `Rc/Weak` 处理树结构。`Vec<T>` + 索引更简单、更快，无引用计数开销。需要动态生命周期的共享所有权时才用 `Rc/Weak`。

----

## Copy vs Clone、PartialEq vs Eq — 何时 derive 什么 {#copy-vs-clone-partialeq-vs-eq--when-to-derive-what}

- **Copy ≈ C++ 可平凡复制（无自定义拷贝构造/析构）。** 如 `int`、枚举、简单 POD 结构体 — 编译器自动生成按位 `memcpy`。Rust 中 `Copy` 同理：赋值 `let b = a;` 隐式按位复制，两变量仍有效。
- **Clone ≈ C++ 拷贝构造 / `operator=` 深拷贝。** C++ 类有自定义拷贝构造（如深拷贝 `std::vector` 成员）时，Rust 等价是实现 `Clone`。必须显式 `.clone()` — Rust 不会在 `=` 后隐藏昂贵拷贝。
- **关键区别：** C++ 中平凡拷贝与深拷贝都通过同一 `=` 语法隐式发生。Rust 强制选择：`Copy` 类型静默复制（廉价），非 `Copy` 默认**移动**，昂贵重复须用 `.clone()` 显式选择。
- 同理，C++ `operator==` 不区分 `a == a` 恒真的类型（如整数）与不恒真的类型（如带 NaN 的 `float`）。Rust 用 `PartialEq` vs `Eq` 编码这一点。

### Copy vs Clone {#copy-vs-clone}

| | **Copy** | **Clone** |
|---|---------|----------|
| **如何工作** | 按位 memcpy（隐式） | 自定义逻辑（显式 `.clone()`） |
| **何时发生** | 赋值：`let b = a;` | 仅调用 `.clone()` 时 |
| **复制/clone 后** | `a` 与 `b` 均有效 | `a` 与 `b` 均有效 |
| **既无 Copy 也无 Clone** | `let b = a;` **移动** `a`（`a` 消失） | `let b = a;` **移动** `a`（`a` 消失） |
| **允许用于** | 无堆数据的类型 | 任意类型 |
| **C++ 类比** | 可平凡复制 / POD（无自定义拷贝构造） | 自定义拷贝构造（深拷贝） |

### 真实示例：Copy — 简单枚举 {#real-example-copy--simple-enums}
```rust
// From fan_diag/src/sensor.rs — all unit variants, fits in 1 byte
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Default)]
pub enum FanStatus {
    #[default]
    Normal,
    Low,
    High,
    Missing,
    Failed,
    Unknown,
}

let status = FanStatus::Normal;
let copy = status;   // Implicit copy — status is still valid
println!("{:?} {:?}", status, copy);  // Both work
```

### 真实示例：Copy — 带整数载荷的枚举 {#real-example-copy--enum-with-integer-payloads}
```rust
// Example: healthcheck.rs — u32 payloads are Copy, so the whole enum is too
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum HealthcheckStatus {
    Pass,
    ProgramError(u32),
    DmesgError(u32),
    RasError(u32),
    OtherError(u32),
    Unknown,
}
```

### 真实示例：仅 Clone — 含堆数据的结构体 {#real-example-clone-only--struct-with-heap-data}
```rust
// Example: components.rs — String prevents Copy
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FruData {
    pub technology: DeviceTechnology,
    pub physical_location: String,      // ← String: heap-allocated, can't Copy
    pub expected: bool,
    pub removable: bool,
}
// let a = fru_data;   → MOVES (a is gone)
// let a = fru_data.clone();  → CLONES (fru_data still valid, new heap allocation)
```

### 规则：能否 Copy？ {#the-rule-can-it-be-copy}
```text
Does the type contain String, Vec, Box, HashMap,
Rc, Arc, or any other heap-owning type?
    YES → Clone only (cannot be Copy)
    NO  → You CAN derive Copy (and should, if the type is small)
```

### PartialEq vs Eq {#partialeq-vs-eq}

| | **PartialEq** | **Eq** |
|---|--------------|-------|
| **提供什么** | `==` 与 `!=` | 标记：「相等是自反的」 |
| **自反？（a == a）** | 不保证 | **保证** |
| **为何重要** | `f32::NAN != f32::NAN` | `HashMap` 键**要求** `Eq` |
| **何时 derive** | 几乎总是 | 类型无 `f32`/`f64` 字段时 |
| **C++ 类比** | `operator==` | 无直接等价（C++ 不检查） |

### 真实示例：Eq — 用作 HashMap 键 {#real-example-eq--used-as-hashmap-key}
```rust
// From hms_trap/src/cpu_handler.rs — Hash requires Eq
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum CpuFaultType {
    InvalidFaultType,
    CpuCperFatalErr,
    CpuLpddr5UceErr,
    CpuC2CUceFatalErr,
    // ...
}
// Used as: HashMap<CpuFaultType, FaultHandler>
// HashMap keys must be Eq + Hash — PartialEq alone won't compile
```

### 真实示例：无法 Eq — 类型含 f32 {#real-example-no-eq-possible--type-contains-f32}
```rust
// Example: types.rs — f32 prevents Eq
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct TemperatureSensors {
    pub warning_threshold: Option<f32>,   // ← f32 has NaN ≠ NaN
    pub critical_threshold: Option<f32>,  // ← can't derive Eq
    pub sensor_names: Vec<String>,
}
// Cannot be used as HashMap key. Cannot derive Eq.
// Because: f32::NAN == f32::NAN is false, violating reflexivity.
```

### PartialOrd vs Ord {#partialord-vs-ord}

| | **PartialOrd** | **Ord** |
|---|---------------|--------|
| **提供什么** | `<`、`>`、`<=`、`>=` | `.sort()`、`BTreeMap` 键 |
| **全序？** | 否（某些对可能不可比） | **是**（每对都可比） |
| **f32/f64？** | 仅 PartialOrd（NaN 破坏序） | 不能 derive Ord |

### 真实示例：Ord — 严重级别排序 {#real-example-ord--severity-ranking}
```rust
// From hms_trap/src/fault.rs — variant order defines severity
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum FaultSeverity {
    Info,      // lowest  (discriminant 0)
    Warning,   //         (discriminant 1)
    Error,     //         (discriminant 2)
    Critical,  // highest (discriminant 3)
}
// FaultSeverity::Info < FaultSeverity::Critical → true
// Enables: if severity >= FaultSeverity::Error { escalate(); }
```

### 真实示例：Ord — 诊断级别比较 {#real-example-ord--diagnostic-levels-for-comparison}
```rust
// Example: orchestration.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Default)]
pub enum GpuDiagLevel {
    #[default]
    Quick,     // lowest
    Standard,
    Extended,
    Full,      // highest
}
// Enables: if requested_level >= GpuDiagLevel::Extended { run_extended_tests(); }
```

### Derive 决策树 {#derive-decision-tree}

```text
                        Your new type
                            │
                   Contains String/Vec/Box?
                      /              \
                    YES                NO
                     │                  │
              Clone only          Clone + Copy
                     │                  │
              Contains f32/f64?    Contains f32/f64?
                /          \         /          \
              YES           NO     YES           NO
               │             │      │             │
         PartialEq       PartialEq  PartialEq  PartialEq
         only            + Eq       only       + Eq
                          │                      │
                    Need sorting?           Need sorting?
                      /       \               /       \
                    YES        NO            YES        NO
                     │          │              │          │
               PartialOrd    Done        PartialOrd    Done
               + Ord                     + Ord
                     │                        │
               Need as                  Need as
               map key?                 map key?
                  │                        │
                + Hash                   + Hash
```

### 快速参考：生产 Rust 中的常见 derive 组合 {#quick-reference-common-derive-combos-from-production-rust-code}

| **类型类别** | **典型 derive** | **示例** |
|-------------------|--------------------|------------|
| 简单状态枚举 | `Copy, Clone, PartialEq, Eq, Default` | `FanStatus` |
| 用作 HashMap 键的枚举 | `Copy, Clone, PartialEq, Eq, Hash` | `CpuFaultType`、`SelComponent` |
| 可排序严重级别枚举 | `Copy, Clone, PartialEq, Eq, PartialOrd, Ord` | `FaultSeverity`、`GpuDiagLevel` |
| 含 String 的数据结构体 | `Clone, Debug, Serialize, Deserialize` | `FruData`、`OverallSummary` |
| 可序列化配置 | `Clone, Debug, Default, Serialize, Deserialize` | `DiagConfig` |

----

