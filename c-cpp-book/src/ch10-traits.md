# Rust Trait（特征） {#rust-traits}

> **你将学到：** Trait——Rust 对接口、抽象基类和运算符重载的回答。你将学习如何定义 Trait、为类型实现 Trait，以及如何使用动态分发（`dyn Trait`）与静态分发（泛型）。对 C++ 开发者：Trait 替代虚函数、CRTP 和 concepts。对 C 开发者：Trait 是 Rust 实现多态的结构化方式。

- Rust Trait 与其他语言中的接口类似
    - Trait 定义必须由实现该 Trait 的类型所定义的方法。
```rust
fn main() {
    trait Pet {
        fn speak(&self);
    }
    struct Cat;
    struct Dog;
    impl Pet for Cat {
        fn speak(&self) {
            println!("Meow");
        }
    }
    impl Pet for Dog {
        fn speak(&self) {
            println!("Woof!")
        }
    }
    let c = Cat{};
    let d = Dog{};
    c.speak();  // There is no "is a" relationship between Cat and Dog
    d.speak(); // There is no "is a" relationship between Cat and Dog
}
```

## Trait 与 C++ Concepts 及接口 {#traits-vs-c-concepts-and-interfaces}

### 传统 C++ 继承与 Rust Trait

```cpp
// C++ - Inheritance-based polymorphism
class Animal {
public:
    virtual void speak() = 0;  // Pure virtual function
    virtual ~Animal() = default;
};

class Cat : public Animal {  // "Cat IS-A Animal"
public:
    void speak() override {
        std::cout << "Meow" << std::endl;
    }
};

void make_sound(Animal* animal) {  // Runtime polymorphism
    animal->speak();  // Virtual function call
}
```

```rust
// Rust - Composition over inheritance with traits
trait Animal {
    fn speak(&self);
}

struct Cat;  // Cat is NOT an Animal, but IMPLEMENTS Animal behavior

impl Animal for Cat {  // "Cat CAN-DO Animal behavior"
    fn speak(&self) {
        println!("Meow");
    }
}

fn make_sound<T: Animal>(animal: &T) {  // Static polymorphism
    animal.speak();  // Direct function call (zero cost)
}
```

```mermaid
graph TD
    subgraph "C++ Object-Oriented Hierarchy"
        CPP_ANIMAL["Animal<br/>(Abstract base class)"]
        CPP_CAT["Cat : public Animal<br/>(IS-A relationship)"]
        CPP_DOG["Dog : public Animal<br/>(IS-A relationship)"]
        
        CPP_ANIMAL --> CPP_CAT
        CPP_ANIMAL --> CPP_DOG
        
        CPP_VTABLE["Virtual function table<br/>(Runtime dispatch)"]
        CPP_HEAP["Often requires<br/>heap allocation"]
        CPP_ISSUES["[ERROR] Deep inheritance trees<br/>[ERROR] Diamond problem<br/>[ERROR] Runtime overhead<br/>[ERROR] Tight coupling"]
    end
    
    subgraph "Rust Trait-Based Composition"
        RUST_TRAIT["trait Animal<br/>(Behavior definition)"]
        RUST_CAT["struct Cat<br/>(Data only)"]
        RUST_DOG["struct Dog<br/>(Data only)"]
        
        RUST_CAT -.->|"impl Animal for Cat<br/>(CAN-DO behavior)"| RUST_TRAIT
        RUST_DOG -.->|"impl Animal for Dog<br/>(CAN-DO behavior)"| RUST_TRAIT
        
        RUST_STATIC["Static dispatch<br/>(Compile-time)"]
        RUST_STACK["Stack allocation<br/>possible"]
        RUST_BENEFITS["[OK] No inheritance hierarchy<br/>[OK] Multiple trait impls<br/>[OK] Zero runtime cost<br/>[OK] Loose coupling"]
    end
    
    style CPP_ISSUES fill:#ff6b6b,color:#000
    style RUST_BENEFITS fill:#91e5a3,color:#000
    style CPP_VTABLE fill:#ffa07a,color:#000
    style RUST_STATIC fill:#91e5a3,color:#000
```

### Trait 约束与泛型约束

```rust
use std::fmt::Display;
use std::ops::Add;

// C++ template equivalent (less constrained)
// template<typename T>
// T add_and_print(T a, T b) {
//     // No guarantee T supports + or printing
//     return a + b;  // Might fail at compile time
// }

// Rust - explicit trait bounds
fn add_and_print<T>(a: T, b: T) -> T 
where 
    T: Display + Add<Output = T> + Copy,
{
    println!("Adding {} + {}", a, b);  // Display trait
    a + b  // Add trait
}
```

```mermaid
graph TD
    subgraph "Generic Constraints Evolution"
        UNCONSTRAINED["fn process<T>(data: T)<br/>[ERROR] T can be anything"]
        SINGLE_BOUND["fn process<T: Display>(data: T)<br/>[OK] T must implement Display"]
        MULTI_BOUND["fn process<T>(data: T)<br/>where T: Display + Clone + Debug<br/>[OK] Multiple requirements"]
        
        UNCONSTRAINED --> SINGLE_BOUND
        SINGLE_BOUND --> MULTI_BOUND
    end
    
    subgraph "Trait Bound Syntax"
        INLINE["fn func<T: Trait>(param: T)"]
        WHERE_CLAUSE["fn func<T>(param: T)<br/>where T: Trait"]
        IMPL_PARAM["fn func(param: impl Trait)"]
        
        COMPARISON["Inline: Simple cases<br/>Where: Complex bounds<br/>impl: Concise syntax"]
    end
    
    subgraph "Compile-time Magic"
        GENERIC_FUNC["Generic function<br/>with trait bounds"]
        TYPE_CHECK["Compiler verifies<br/>trait implementations"]
        MONOMORPH["Monomorphization<br/>(Create specialized versions)"]
        OPTIMIZED["Fully optimized<br/>machine code"]
        
        GENERIC_FUNC --> TYPE_CHECK
        TYPE_CHECK --> MONOMORPH
        MONOMORPH --> OPTIMIZED
        
        EXAMPLE["add_and_print::<i32><br/>add_and_print::<f64><br/>(Separate functions generated)"]
        MONOMORPH --> EXAMPLE
    end
    
    style UNCONSTRAINED fill:#ff6b6b,color:#000
    style SINGLE_BOUND fill:#ffa07a,color:#000
    style MULTI_BOUND fill:#91e5a3,color:#000
    style OPTIMIZED fill:#91e5a3,color:#000
```

### C++ 运算符重载 → Rust `std::ops` Trait {#c-operator-overloading--rust-stdops-traits}

在 C++ 中，通过编写具有特殊名称（`operator+`、`operator<<`、`operator[]` 等）的自由函数或成员函数来重载运算符。在 Rust 中，每个运算符都映射到 `std::ops`（或用于输出的 `std::fmt`）中的 Trait。你需要**实现 Trait**，而不是编写魔法命名的函数。

#### 并排对比：`+` 运算符

```cpp
// C++: operator overloading as a member or free function
struct Vec2 {
    double x, y;
    Vec2 operator+(const Vec2& rhs) const {
        return {x + rhs.x, y + rhs.y};
    }
};

Vec2 a{1.0, 2.0}, b{3.0, 4.0};
Vec2 c = a + b;  // calls a.operator+(b)
```

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {
    type Output = Vec2;                     // Associated type — the result of +
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

let a = Vec2 { x: 1.0, y: 2.0 };
let b = Vec2 { x: 3.0, y: 4.0 };
let c = a + b;  // calls <Vec2 as Add>::add(a, b)
println!("{c:?}"); // Vec2 { x: 4.0, y: 6.0 }
```

#### 与 C++ 的关键差异

| 方面 | C++ | Rust |
|--------|-----|------|
| **机制** | 魔法函数名（`operator+`） | 实现 Trait（`impl Add for T`） |
| **发现性** | 搜索 `operator+` 或阅读头文件 | 查看 Trait 实现——IDE 支持极佳 |
| **返回类型** | 自由选择 | 由 `Output` 关联类型固定 |
| **接收者** | 通常接受 `const T&`（借用） | 默认按值接受 `self`（移动！） |
| **对称性** | 可写 `impl operator+(int, Vec2)` | 须添加 `impl Add<Vec2> for i32`（适用孤儿规则） |
| **用 `<<` 打印** | `operator<<(ostream&, T)`——可为*任意*流重载 | `impl fmt::Display for T`——一种规范的 `to_string` 表示 |

#### 按值 `self` 的陷阱

在 Rust 中，`Add::add(self, rhs)` 按值接受 `self`。**对于 `Copy` 类型**（如上文的 `Vec2`，它 derive 了 `Copy`）这没问题——编译器会复制。但对于非 `Copy` 类型，`+` 会**消耗**操作数：

```rust
let s1 = String::from("hello ");
let s2 = String::from("world");
let s3 = s1 + &s2;  // s1 is MOVED into s3!
// println!("{s1}");  // ❌ Compile error: value used after move
println!("{s2}");     // ✅ s2 was only borrowed (&s2)
```

这就是为什么 `String + &str` 可行而 `&str + &str` 不行——`Add` 只为 `String + &str` 实现，消耗左侧的 `String` 以复用其缓冲区。C++ 中没有对应物：`std::string::operator+` 总是创建新字符串。

#### 完整映射：C++ 运算符 → Rust Trait

| C++ 运算符 | Rust Trait | 说明 |
|-------------|-----------|-------|
| `operator+` | `std::ops::Add` | `Output` 关联类型 |
| `operator-` | `std::ops::Sub` | |
| `operator*` | `std::ops::Mul` | 不是指针解引用——那是 `Deref` |
| `operator/` | `std::ops::Div` | |
| `operator%` | `std::ops::Rem` | |
| `operator-`（一元） | `std::ops::Neg` | |
| `operator!` / `operator~` | `std::ops::Not` | Rust 用 `!` 表示逻辑和按位 NOT（没有 `~` 运算符） |
| `operator&`, `\|`, `^` | `BitAnd`, `BitOr`, `BitXor` | |
| `operator<<`, `>>`（移位） | `Shl`, `Shr` | 不是流 I/O！ |
| `operator+=` | `std::ops::AddAssign` | 接受 `&mut self`（不是 `self`） |
| `operator[]` | `std::ops::Index` / `IndexMut` | 返回 `&Output` / `&mut Output` |
| `operator()` | `Fn` / `FnMut` / `FnOnce` | 闭包实现这些；不能直接 `impl Fn` |
| `operator==` | `PartialEq`（+ `Eq`） | 在 `std::cmp` 中，不在 `std::ops` |
| `operator<` | `PartialOrd`（+ `Ord`） | 在 `std::cmp` |
| `operator<<`（流） | `fmt::Display` | `println!("{}", x)` |
| `operator<<`（调试） | `fmt::Debug` | `println!("{:?}", x)` |
| `operator bool` | 无直接等价物 | 使用 `impl From<T> for bool` 或 `.is_empty()` 等命名方法 |
| `operator T()`（隐式转换） | 无隐式转换 | 使用 `From`/`Into` Trait（显式） |

#### 护栏：Rust 阻止了什么

1. **无隐式转换**：C++ 的 `operator int()` 可能导致静默、令人惊讶的强制转换。Rust 没有隐式转换运算符——使用 `From`/`Into` 并显式调用 `.into()`。
2. **不能重载 `&&` / `||`**：C++ 允许（破坏短路语义！）。Rust 不允许。
3. **不能重载 `=`**：赋值始终是移动或复制，从不是用户定义的。复合赋值（`+=`）可通过 `AddAssign` 等重载。
4. **不能重载 `,`**：C++ 允许 `operator,()`——最著名的 C++ 陷阱之一。Rust 不允许。
5. **不能重载 `&`（取地址）**：又一个 C++ 陷阱（`std::addressof` 的存在就是为了绕过它）。Rust 的 `&` 始终表示「借用」。
6. **一致性规则**：你只能为自有类型实现 `Add<Foreign>`，或为外部类型实现 `Add<YourType>`——永远不能为 `Foreign` 实现 `Add<Foreign>`。这防止跨 crate 的冲突运算符定义。

> **结论**：在 C++ 中，运算符重载功能强大但基本不受约束——你几乎可以重载任何东西，包括逗号和取地址，隐式转换可以静默触发。Rust 通过 Trait 为算术和比较运算符提供同样的表达能力，但**阻止历史上危险的重载**，并强制所有转换显式进行。

----
# Rust Trait（特征）
- Rust 允许在甚至像本例中 `u32` 这样的内置类型上实现用户定义的 Trait。然而，Trait 或类型必须属于该 crate
```rust
trait IsSecret {
  fn is_secret(&self);
}
// The IsSecret trait belongs to the crate, so we are OK
impl IsSecret for u32 {
  fn is_secret(&self) {
      if *self == 42 {
          println!("Is secret of life");
      }
  }
}

fn main() {
  42u32.is_secret();
  43u32.is_secret();
}
```


# Rust Trait（特征）
- Trait 支持接口继承和默认实现
```rust
trait Animal {
  // Default implementation
  fn is_mammal(&self) -> bool {
    true
  }
}
trait Feline : Animal {
  // Default implementation
  fn is_feline(&self) -> bool {
    true
  }
}

struct Cat;
// Use default implementations. Note that all traits for the supertrait must be individually implemented
impl Feline for Cat {}
impl Animal for Cat {}
fn main() {
  let c = Cat{};
  println!("{} {}", c.is_mammal(), c.is_feline());
}
```
----
# 练习：Logger Trait 实现 {#exercise-logger-trait-implementation}

🟡 **进阶**

- 实现一个名为 `log()`、接受 `u64` 的 ```Log trait```
    - 实现两个不同的 logger ```SimpleLogger``` 和 ```ComplexLogger```，它们实现 ```Log trait```。一个应输出 "Simple logger" 及 ```u64```，另一个应输出 "Complex logger" 及 ```u64```

<details><summary>Solution (click to expand)</summary>

```rust
trait Log {
    fn log(&self, value: u64);
}

struct SimpleLogger;
struct ComplexLogger;

impl Log for SimpleLogger {
    fn log(&self, value: u64) {
        println!("Simple logger: {value}");
    }
}

impl Log for ComplexLogger {
    fn log(&self, value: u64) {
        println!("Complex logger: {value} (hex: 0x{value:x}, binary: {value:b})");
    }
}

fn main() {
    let simple = SimpleLogger;
    let complex = ComplexLogger;
    simple.log(42);
    complex.log(42);
}
// Output:
// Simple logger: 42
// Complex logger: 42 (hex: 0x2a, binary: 101010)
```

</details>

----
# Rust Trait 关联类型
```rust
#[derive(Debug)]
struct Small(u32);
#[derive(Debug)]
struct Big(u32);
trait Double {
    type T;
    fn double(&self) -> Self::T;
}

impl Double for Small {
    type T = Big;
    fn double(&self) -> Self::T {
        Big(self.0 * 2)
    }
}
fn main() {
    let a = Small(42);
    println!("{:?}", a.double());
}
```

# Rust Trait impl
- ```impl``` 可与 Trait 一起使用，以接受任何实现了 Trait 的类型
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &impl Pet) {
    p.speak();
}
fn main() {
    let c = Cat {};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```

# Rust Trait impl
- ```impl``` 也可用于返回值
```rust
trait Pet {}
struct Dog;
struct Cat;
impl Pet for Cat {}
impl Pet for Dog {}
fn cat_as_pet() -> impl Pet {
    let c = Cat {};
    c
}
fn dog_as_pet() -> impl Pet {
    let d = Dog {};
    d
}
fn main() {
    let p = cat_as_pet();
    let d = dog_as_pet();
}
```
----
# Rust 动态 Trait
- 动态 Trait 可用于在不知道底层类型的情况下调用 Trait 功能。这称为 ```type erasure```
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {x: u32}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &dyn Pet) {
    p.speak();
}
fn main() {
    let c = Cat {x: 42};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```
----

## 何时使用 enum 与 dyn Trait {#when-to-use-enum-vs-dyn-trait}

这三种方式都能实现多态，但权衡不同：

| 方式 | 分发 | 性能 | 异构集合？ | 何时使用 |
|----------|----------|-------------|---------------------------|-------------|
| `impl Trait` / 泛型 | 静态（单态化） | 零成本——编译期内联 | 否——每个槽位只有一种具体类型 | 默认选择。函数参数、返回类型 |
| `dyn Trait` | 动态（vtable） | 每次调用略有开销（约 1 次指针间接） | 是——`Vec<Box<dyn Trait>>` | 需要在集合中混合类型，或插件式可扩展性 |
| `enum` | match | 零成本——编译期已知变体 | 是——但仅限已知变体 | 变体集合**封闭**且在编译期已知 |

```rust
trait Shape {
    fn area(&self) -> f64;
}
struct Circle { radius: f64 }
struct Rect { w: f64, h: f64 }
impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius } }
impl Shape for Rect   { fn area(&self) -> f64 { self.w * self.h } }

// Static dispatch — compiler generates separate code for each type
fn print_area(s: &impl Shape) { println!("{}", s.area()); }

// Dynamic dispatch — one function, works with any Shape behind a pointer
fn print_area_dyn(s: &dyn Shape) { println!("{}", s.area()); }

// Enum — closed set, no trait needed
enum ShapeEnum { Circle(f64), Rect(f64, f64) }
impl ShapeEnum {
    fn area(&self) -> f64 {
        match self {
            ShapeEnum::Circle(r) => std::f64::consts::PI * r * r,
            ShapeEnum::Rect(w, h) => w * h,
        }
    }
}
```

> **对 C++ 开发者：** `impl Trait` 类似 C++ 模板（单态化、零成本）。`dyn Trait` 类似 C++ 虚函数（vtable 分发）。带 `match` 的 Rust 枚举类似 `std::variant` 配合 `std::visit`——但穷尽匹配由编译器强制。

> **经验法则**：从 `impl Trait`（静态分发）开始。仅当需要异构集合或编译期无法知道具体类型时才用 `dyn Trait`。当你拥有所有变体时用 `enum`。

