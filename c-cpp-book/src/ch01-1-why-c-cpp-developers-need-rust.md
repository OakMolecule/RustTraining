# 为什么 C/C++ 开发者需要 Rust

> **你将学到：**
> - Rust 消除的完整问题清单——内存安全、未定义行为（Undefined Behavior）、数据竞争等
> - 为什么 `shared_ptr`、`unique_ptr` 等 C++ 缓解手段只是创可贴，而非根本解决方案
> - 在 safe Rust 中结构上不可能出现的具体 C/C++ 漏洞示例

> **想直接看代码？** 跳转到 [给我看代码](ch02-getting-started.md#enough-talk-already-show-me-some-code)

## Rust 消除了什么——完整清单 {#what-rust-eliminates--the-complete-list}

在深入示例之前，先给出执行摘要。Safe Rust **从结构上防止** 本清单中的每一个问题——不是靠纪律、工具或代码审查，而是靠类型系统和编译器：

| **被消除的问题** | **C** | **C++** | **Rust 如何防止** |
|----------------------|:-----:|:-------:|--------------------------|
| 缓冲区溢出 / 下溢 | ✅ | ✅ | 所有数组、切片和字符串都携带边界信息；索引在运行时检查 |
| 内存泄漏（无需 GC） | ✅ | ✅ | `Drop` Trait（特征）= 正确实现的 RAII；自动清理，无需 Rule of Five |
| 悬垂指针 | ✅ | ✅ | 生命周期（Lifetime）系统在编译期证明引用比其指向对象活得更久 |
| 释放后使用（Use-after-free） | ✅ | ✅ | 所有权（Ownership）系统使其成为编译错误 |
| 移动后使用（Use-after-move） | — | ✅ | 移动是**破坏性的**——原绑定不再存在 |
| 未初始化变量 | ✅ | ✅ | 所有变量使用前必须初始化；编译器强制检查 |
| 整数溢出 / 下溢 UB | ✅ | ✅ | Debug 构建在溢出时 panic；Release 构建环绕（两种方式均为定义明确的行为） |
| NULL 指针解引用 / SEGV | ✅ | ✅ | 没有空指针；`Option<T>` 强制显式处理 |
| 数据竞争 | ✅ | ✅ | `Send`/`Sync` Trait + 借用检查器使数据竞争成为编译错误 |
| 不受控的副作用 | ✅ | ✅ | 默认不可变；修改需要显式 `mut` |
| 无继承（更好的可维护性） | — | ✅ | Trait + 组合替代类层次结构；促进复用而不产生耦合 |
| 无异常；可预测的控制流 | — | ✅ | 错误是值（`Result<T, E>`）；无法忽略，没有隐藏的 `throw` 路径 |
| 迭代器失效 | — | ✅ | 借用检查器禁止在迭代时修改集合 |
| 引用循环 / 终结器泄漏 | — | ✅ | 所有权呈树形结构；`Rc` 循环是可选的，可用 `Weak` 检测 |
| 不会忘记解锁互斥锁 | ✅ | ✅ | `Mutex<T>` 包装数据；锁守卫是访问数据的唯一方式 |
| 未定义行为（一般情况） | ✅ | ✅ | Safe Rust **零**未定义行为；`unsafe` 代码块显式且可审计 |

> **结论：** 这些不是靠编码规范追求的理想目标。它们是**编译期保证**。只要代码能编译通过，这些 bug 就不可能存在。

---

## C 与 C++ 共同面临的问题 {#the-problems-shared-by-c-and-c}

> **想跳过示例？** 跳转到 [Rust 如何解决这一切](#how-rust-addresses-all-of-this)，或直接前往 [给我看代码](ch02-getting-started.md#enough-talk-already-show-me-some-code)

两种语言共享一组核心内存安全问题，它们是超过 70% CVE（Common Vulnerabilities and Exposures，常见漏洞与披露）的根本原因：

### 缓冲区溢出

C 的数组、指针和字符串没有内在边界。越界极其容易：

```c
#include <stdlib.h>
#include <string.h>

void buffer_dangers() {
    char buffer[10];
    strcpy(buffer, "This string is way too long!");  // Buffer overflow

    int arr[5] = {1, 2, 3, 4, 5};
    int *ptr = arr;           // Loses size information
    ptr[10] = 42;             // No bounds check — undefined behavior
}
```

在 C++ 中，`std::vector::operator[]` 仍不进行边界检查。只有 `.at()` 会检查——但谁会去捕获那个异常？

### 悬垂指针与释放后使用

```c
int *bar() {
    int i = 42;
    return &i;    // Returns address of stack variable — dangling!
}

void use_after_free() {
    char *p = (char *)malloc(20);
    free(p);
    *p = '\0';   // Use after free — undefined behavior
}
```

### 未初始化变量与未定义行为

C 和 C++ 都允许未初始化变量。产生的值是不确定的，读取它们是未定义行为：

```c
int x;               // Uninitialized
if (x > 0) { ... }  // UB — x could be anything
```

对于无符号类型，C 中整数溢出是**定义明确的**；对于有符号类型则是**未定义的**。在 C++ 中，有符号溢出同样是未定义行为。两种编译器都会利用这一点做"优化"，以出人意料的方式破坏程序。

### NULL 指针解引用

```c
int *ptr = NULL;
*ptr = 42;           // SEGV — but the compiler won't stop you
```

在 C++ 中，`std::optional<T>` 有帮助，但写法冗长，且常被 `.value()` 绕过——后者会抛出异常。

### 可视化：共同问题

```mermaid
graph TD
    ROOT["C/C++ Memory Safety Issues"] --> BUF["Buffer Overflows"]
    ROOT --> DANGLE["Dangling Pointers"]
    ROOT --> UAF["Use-After-Free"]
    ROOT --> UNINIT["Uninitialized Variables"]
    ROOT --> NULL["NULL Dereferences"]
    ROOT --> UB["Undefined Behavior"]
    ROOT --> RACE["Data Races"]

    BUF --> BUF1["No bounds on arrays/pointers"]
    DANGLE --> DANGLE1["Returning stack addresses"]
    UAF --> UAF1["Reusing freed memory"]
    UNINIT --> UNINIT1["Indeterminate values"]
    NULL --> NULL1["No forced null checks"]
    UB --> UB1["Signed overflow, aliasing"]
    RACE --> RACE1["No compile-time safety"]

    style ROOT fill:#ff6b6b,color:#000
    style BUF fill:#ffa07a,color:#000
    style DANGLE fill:#ffa07a,color:#000
    style UAF fill:#ffa07a,color:#000
    style UNINIT fill:#ffa07a,color:#000
    style NULL fill:#ffa07a,color:#000
    style UB fill:#ffa07a,color:#000
    style RACE fill:#ffa07a,color:#000
```

---

## C++ 在此基础上又增加了更多问题 {#c-adds-more-problems-on-top}

> **C 读者**：如果不使用 C++，可以[跳到 Rust 如何解决这些问题](#how-rust-addresses-all-of-this)。
>
> **想直接看代码？** 跳转到 [给我看代码](ch02-getting-started.md#enough-talk-already-show-me-some-code)

C++ 引入了智能指针、RAII、移动语义和异常来解决 C 的问题。但这些只是**创可贴，不是根治**——它们把失败模式从"运行时崩溃"变成了"运行时更隐蔽的 bug"：

### `unique_ptr` 与 `shared_ptr`——创可贴，而非解决方案

C++ 智能指针相比原始 `malloc`/`free` 是显著改进，但并未解决底层问题：

| C++ 缓解手段 | 解决了什么 | **没解决什么** |
|----------------|---------------|------------------------|
| `std::unique_ptr` | 通过 RAII 防止泄漏 | **移动后使用**仍能编译通过；留下僵尸 nullptr |
| `std::shared_ptr` | 共享所有权 | **引用循环**会静默泄漏；`weak_ptr` 纪律需手动维护 |
| `std::optional` | 替代部分空值用法 | `.value()` 在为空时**抛出异常**——隐藏的控制流 |
| `std::string_view` | 避免拷贝 | 若源字符串被释放则**悬垂**——无生命周期检查 |
| 移动语义 | 高效转移 | 被移动的对象处于**"有效但未指定状态"**——随时可能 UB |
| RAII | 自动清理 | 需要正确实现 **Rule of Five**；一处失误全盘皆输 |

```cpp
// unique_ptr: use-after-move compiles cleanly
std::unique_ptr<int> ptr = std::make_unique<int>(42);
std::unique_ptr<int> ptr2 = std::move(ptr);
std::cout << *ptr;  // Compiles! Undefined behavior at runtime.
                     // In Rust, this is a compile error: "value used after move"
```

```cpp
// shared_ptr: reference cycles leak silently
struct Node {
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> parent;  // Cycle! Destructor never called.
};
auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b;
b->parent = a;  // Memory leak — ref count never reaches 0
                 // In Rust, Rc<T> + Weak<T> makes cycles explicit and breakable
```

### 移动后使用——无声的杀手

C++ 的 `std::move` 不是移动——它是类型转换。原对象仍处于"有效但未指定状态"。编译器允许你继续使用它：

```cpp
auto vec = std::make_unique<std::vector<int>>({1, 2, 3});
auto vec2 = std::move(vec);
vec->size();  // Compiles! But dereferencing nullptr — crash at runtime
```

在 Rust 中，移动是**破坏性的**。原绑定已不存在：

```rust
let vec = vec![1, 2, 3];
let vec2 = vec;           // Move — vec is consumed
// vec.len();             // Compile error: value used after move
```

### 迭代器失效——来自生产环境 C++ 的真实 bug

这些不是牵强附会的例子——它们代表在大型 C++ 代码库中发现的**真实 bug 模式**：

```cpp
// BUG 1: erase without reassigning iterator (undefined behavior)
while (it != pending_faults.end()) {
    if (*it != nullptr && (*it)->GetId() == fault->GetId()) {
        pending_faults.erase(it);   // ← iterator invalidated!
        removed_count++;            //   next loop uses dangling iterator
    } else {
        ++it;
    }
}
// Fix: it = pending_faults.erase(it);
```

```cpp
// BUG 2: index-based erase skips elements
for (auto i = 0; i < entries.size(); i++) {
    if (config_status == ConfigDisable::Status::Disabled) {
        entries.erase(entries.begin() + i);  // ← shifts elements
    }                                         //   i++ skips the shifted one
}
```

```cpp
// BUG 3: one erase path correct, the other isn't
while (it != incomplete_ids.end()) {
    if (current_action == nullptr) {
        incomplete_ids.erase(it);  // ← BUG: iterator not reassigned
        continue;
    }
    it = incomplete_ids.erase(it); // ← Correct path
}
```

**这些代码编译时没有任何警告。** 在 Rust 中，借用检查器使以上三种情况都成为编译错误——你无法在迭代集合的同时修改它，绝无例外。

### 异常安全与 `dynamic_cast`/`new` 模式

现代 C++ 代码库仍大量依赖没有编译期安全的模式：

```cpp
// Typical C++ factory pattern — every branch is a potential bug
DriverBase* driver = nullptr;
if (dynamic_cast<ModelA*>(device)) {
    driver = new DriverForModelA(framework);
} else if (dynamic_cast<ModelB*>(device)) {
    driver = new DriverForModelB(framework);
}
// What if driver is still nullptr? What if new throws? Who owns driver?
```

在一个典型的 10 万行 C++ 代码库中，你可能发现数百次 `dynamic_cast` 调用（每次都是潜在的运行时失败）、数百次原始 `new` 调用（每次都是潜在的泄漏），以及数百个 `virtual`/`override` 方法（到处都有 vtable 开销）。

### 悬垂引用与 lambda 捕获

```cpp
int& get_reference() {
    int x = 42;
    return x;  // Dangling reference — compiles, UB at runtime
}

auto make_closure() {
    int local = 42;
    return [&local]() { return local; };  // Dangling capture!
}
```

### 可视化：C++ 额外问题

```mermaid
graph TD
    ROOT["C++ Additional Problems<br/>(on top of C issues)"] --> UAM["Use-After-Move"]
    ROOT --> CYCLE["Reference Cycles"]
    ROOT --> ITER["Iterator Invalidation"]
    ROOT --> EXC["Exception Safety"]
    ROOT --> TMPL["Template Error Messages"]

    UAM --> UAM1["std::move leaves zombie<br/>Compiles without warning"]
    CYCLE --> CYCLE1["shared_ptr cycles leak<br/>Destructor never called"]
    ITER --> ITER1["erase() invalidates iterators<br/>Real production bugs"]
    EXC --> EXC1["Partial construction<br/>new without try/catch"]
    TMPL --> TMPL1["30+ lines of nested<br/>template instantiation errors"]

    style ROOT fill:#ff6b6b,color:#000
    style UAM fill:#ffa07a,color:#000
    style CYCLE fill:#ffa07a,color:#000
    style ITER fill:#ffa07a,color:#000
    style EXC fill:#ffa07a,color:#000
    style TMPL fill:#ffa07a,color:#000
```

---

## Rust 如何解决这一切 {#how-rust-addresses-all-of-this}

上文列出的每一个问题——无论来自 C 还是 C++——都被 Rust 的编译期保证所防止：

| 问题 | Rust 的解决方案 |
|---------|-----------------|
| 缓冲区溢出 | 切片携带长度；索引进行边界检查 |
| 悬垂指针 / 释放后使用 | 生命周期系统在编译期证明引用有效 |
| 移动后使用 | 移动是破坏性的——编译器拒绝让你再触碰原值 |
| 内存泄漏 | `Drop` Trait = 无需 Rule of Five 的 RAII；自动、正确的清理 |
| 引用循环 | 所有权呈树形结构；`Rc` + `Weak` 使循环显式化 |
| 迭代器失效 | 借用检查器禁止在借用集合时修改它 |
| NULL 指针 | 没有 null。`Option<T>` 通过模式匹配强制显式处理 |
| 数据竞争 | `Send`/`Sync` Trait 使数据竞争成为编译错误 |
| 未初始化变量 | 所有变量必须初始化；编译器强制检查 |
| 整数 UB | Debug 在溢出时 panic；Release 环绕（均为定义明确的行为） |
| 异常 | 没有异常；`Result<T, E>` 在类型签名中可见，用 `?` 传播 |
| 继承复杂性 | Trait + 组合；没有菱形问题（Diamond Problem），没有 vtable 脆弱性 |
| 忘记解锁互斥锁 | `Mutex<T>` 包装数据；锁守卫是唯一的访问路径 |

```rust
fn rust_prevents_everything() {
    // ✅ No buffer overflow — bounds checked
    let arr = [1, 2, 3, 4, 5];
    // arr[10];  // panic at runtime, never UB

    // ✅ No use-after-move — compile error
    let data = vec![1, 2, 3];
    let moved = data;
    // data.len();  // error: value used after move

    // ✅ No dangling pointer — lifetime error
    // let r;
    // { let x = 5; r = &x; }  // error: x does not live long enough

    // ✅ No null — Option forces handling
    let maybe: Option<i32> = None;
    // maybe.unwrap();  // panic, but you'd use match or if let instead

    // ✅ No data race — compile error
    // let mut shared = vec![1, 2, 3];
    // std::thread::spawn(|| shared.push(4));  // error: closure may outlive
    // shared.push(5);                         //   borrowed value
}
```

### Rust 的安全模型——全貌

```mermaid
graph TD
    RUST["Rust Safety Guarantees"] --> OWN["Ownership System"]
    RUST --> BORROW["Borrow Checker"]
    RUST --> TYPES["Type System"]
    RUST --> TRAITS["Send/Sync Traits"]

    OWN --> OWN1["No use-after-free<br/>No use-after-move<br/>No double-free"]
    BORROW --> BORROW1["No dangling references<br/>No iterator invalidation<br/>No data races through refs"]
    TYPES --> TYPES1["No NULL (Option&lt;T&gt;)<br/>No exceptions (Result&lt;T,E&gt;)<br/>No uninitialized values"]
    TRAITS --> TRAITS1["No data races<br/>Send = safe to transfer<br/>Sync = safe to share"]

    style RUST fill:#51cf66,color:#000
    style OWN fill:#91e5a3,color:#000
    style BORROW fill:#91e5a3,color:#000
    style TYPES fill:#91e5a3,color:#000
    style TRAITS fill:#91e5a3,color:#000
```

## 快速参考：C vs C++ vs Rust

| **概念** | **C** | **C++** | **Rust** | **关键差异** |
|-------------|-------|---------|----------|-------------------|
| 内存管理 | `malloc()/free()` | `unique_ptr`, `shared_ptr` | `Box<T>`, `Rc<T>`, `Arc<T>` | 自动管理，无循环，无僵尸对象 |
| 数组 | `int arr[10]` | `std::vector<T>`, `std::array<T>` | `Vec<T>`, `[T; N]` | 默认进行边界检查 |
| 字符串 | `char*` 带 `\0` | `std::string`, `string_view` | `String`, `&str` | 保证 UTF-8，生命周期检查 |
| 引用 | `int*`（原始指针） | `T&`, `T&&`（移动） | `&T`, `&mut T` | 生命周期 + 借用检查 |
| 多态 | 函数指针 | 虚函数、继承 | Trait、trait 对象 | 组合优于继承 |
| 泛型 | 宏 / `void*` | 模板 | 泛型 + Trait 约束 | 清晰的错误信息 |
| 错误处理 | 返回码、`errno` | 异常、`std::optional` | `Result<T, E>`, `Option<T>` | 无隐藏控制流 |
| NULL 安全 | `ptr == NULL` | `nullptr`, `std::optional<T>` | `Option<T>` | 强制空值检查 |
| 线程安全 | 手动（pthreads） | 手动（`std::mutex` 等） | 编译期 `Send`/`Sync` | 数据竞争不可能发生 |
| 构建系统 | Make、CMake | CMake、Make 等 | Cargo | 集成工具链 |
| 未定义行为 | 泛滥 | 隐蔽（有符号溢出、别名） | Safe 代码中为零 | 安全有保证 |

***
