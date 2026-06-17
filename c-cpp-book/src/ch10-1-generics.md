# Rust 泛型 {#rust-generics}

> **你将学到：** 泛型类型参数、单态化（零成本泛型）、Trait 约束，以及 Rust 泛型与 C++ 模板的对比——错误信息更清晰，无需 SFINAE。

- 泛型允许同一算法或数据结构在不同数据类型间复用
    - 泛型参数出现在 ```<>``` 内的标识符中，例如：```<T>```。参数可以是任何合法标识符名，但通常保持简短
    - 编译器在编译期执行单态化，即为遇到的 ```T``` 的每种变体生成新类型
```rust
// Returns a tuple of type <T> composed of left and right of type <T>
fn pick<T>(x: u32, left: T, right: T) -> (T, T) {
   if x == 42 {
    (left, right) 
   } else {
    (right, left)
   }
}
fn main() {
    let a = pick(42, true, false);
    let b = pick(42, "hello", "world");
    println!("{a:?}, {b:?}");
}
```

# Rust 泛型
- 泛型也可应用于数据类型及其关联方法。可以为特定的 ```<T>``` 特化实现（示例：```f32``` 与 ```u32```）
```rust
#[derive(Debug)] // We will discuss this later
struct Point<T> {
    x : T,
    y : T,
}
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point {x, y}
    }
    fn set_x(&mut self, x: T) {
         self.x = x;       
    }
    fn set_y(&mut self, y: T) {
         self.y = y;       
    }
}
impl Point<f32> {
    fn is_secret(&self) -> bool {
        self.x == 42.0
    }    
}
fn main() {
    let mut p = Point::new(2, 4); // i32
    let q = Point::new(2.0, 4.0); // f32
    p.set_x(42);
    p.set_y(43);
    println!("{p:?} {q:?} {}", q.is_secret());
}
```

# 练习：泛型 {#exercise-generics}

🟢 **入门**
- 修改 ```Point``` 类型，使 x 和 y 使用两种不同类型（```T``` 和 ```U```）

<details><summary>Solution (click to expand)</summary>

```rust
#[derive(Debug)]
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn new(x: T, y: U) -> Self {
        Point { x, y }
    }
}

fn main() {
    let p1 = Point::new(42, 3.14);        // Point<i32, f64>
    let p2 = Point::new("hello", true);   // Point<&str, bool>
    let p3 = Point::new(1u8, 1000u64);    // Point<u8, u64>
    println!("{p1:?}");
    println!("{p2:?}");
    println!("{p3:?}");
}
// Output:
// Point { x: 42, y: 3.14 }
// Point { x: "hello", y: true }
// Point { x: 1, y: 1000 }
```

</details>

### 结合 Rust Trait 与泛型 {#combining-rust-traits-and-generics}
- Trait 可用于对泛型类型施加限制（约束）
- 约束可在泛型类型参数后用 ```:``` 指定，或使用 ```where```。以下定义泛型函数 ```get_area```，接受任何实现了 ```ComputeArea``` ```trait``` 的类型 ```T```
```rust
    trait ComputeArea {
        fn area(&self) -> u64;
    }
    fn get_area<T: ComputeArea>(t: &T) -> u64 {
        t.area()
    }
```
- [▶ 在 Rust Playground 中尝试](https://play.rust-lang.org/)

### 结合 Rust Trait 与泛型
- 可以有多个 Trait 约束
```rust
trait Fish {}
trait Mammal {}
struct Shark;
struct Whale;
impl Fish for Shark {}
impl Fish for Whale {}
impl Mammal for Whale {}
fn only_fish_and_mammals<T: Fish + Mammal>(_t: &T) {}
fn main() {
    let w = Whale {};
    only_fish_and_mammals(&w);
    let _s = Shark {};
    // Won't compile
    only_fish_and_mammals(&_s);
}
```

### 数据类型中的 Rust Trait 约束 {#rust-traits-constraints-in-data-types}
- Trait 约束可与泛型结合用于数据类型
- 在以下示例中，我们定义 ```PrintDescription``` ```trait``` 和泛型 ```struct``` ```Shape```，其成员受 Trait 约束
```rust
trait PrintDescription {
    fn print_description(&self);
}
struct Shape<S: PrintDescription> {
    shape: S,
}
// Generic Shape implementation for any type that implements PrintDescription
impl<S: PrintDescription> Shape<S> {
    fn print(&self) {
        self.shape.print_description();
    }
}
```
- [▶ 在 Rust Playground 中尝试](https://play.rust-lang.org/)

# 练习：Trait 约束与泛型 {#exercise-trait-constraints-and-generics}

🟡 **进阶**
- 实现一个带有泛型成员 ```cipher```、且该成员实现 ```CipherText``` 的 ```struct```
```rust
trait CipherText {
    fn encrypt(&self);
}
// TO DO
//struct Cipher<>

```
- 接下来，在 ```struct``` 的 ```impl``` 上实现名为 ```encrypt``` 的方法，调用 ```cipher``` 上的 ```encrypt```
```rust
// TO DO
impl for Cipher<> {}
```
- 接下来，在两个名为 ```CipherOne``` 和 ```CipherTwo``` 的结构体上实现 ```CipherText```（```println()``` 即可）。创建 ```CipherOne``` 和 ```CipherTwo```，并用 ```Cipher``` 调用它们

<details><summary>Solution (click to expand)</summary>

```rust
trait CipherText {
    fn encrypt(&self);
}

struct Cipher<T: CipherText> {
    cipher: T,
}

impl<T: CipherText> Cipher<T> {
    fn encrypt(&self) {
        self.cipher.encrypt();
    }
}

struct CipherOne;
struct CipherTwo;

impl CipherText for CipherOne {
    fn encrypt(&self) {
        println!("CipherOne encryption applied");
    }
}

impl CipherText for CipherTwo {
    fn encrypt(&self) {
        println!("CipherTwo encryption applied");
    }
}

fn main() {
    let c1 = Cipher { cipher: CipherOne };
    let c2 = Cipher { cipher: CipherTwo };
    c1.encrypt();
    c2.encrypt();
}
// Output:
// CipherOne encryption applied
// CipherTwo encryption applied
```

</details>

### Rust 类型状态模式与泛型 {#rust-type-state-pattern-and-generics}
- Rust 类型可在*编译期*强制状态机转换
    - 考虑一个 ```Drone```，假设有两种状态：```Idle``` 和 ```Flying```。在 ```Idle``` 状态下，唯一允许的方法是 ```takeoff()```。在 ```Flying``` 状态下，我们允许 ```land()```
    
- 一种方法是用类似以下的方式建模状态机
```rust
enum DroneState {
    Idle,
    Flying
}
struct Drone {x: u64, y: u64, z: u64, state: DroneState}  // x, y, z are coordinates
```
- 这需要大量运行时检查来强制状态机语义——[▶ 试试看](https://play.rust-lang.org/) 了解原因

### Rust 类型状态模式泛型
- 泛型允许我们在*编译期*强制状态机。这需要使用名为 ```PhantomData<T>``` 的特殊泛型
- ```PhantomData<T>``` 是 ```zero-sized``` 标记数据类型。此处我们用它表示 ```Idle``` 和 ```Flying``` 状态，但运行时大小为 ```zero```
- 注意 ```takeoff``` 和 ```land``` 方法将 ```self``` 作为参数。这称为 ```consuming```（与使用借用的 ```&self``` 相对）。基本上，一旦我们在 ```Drone<Idle>``` 上调用 ```takeoff()```，只能得到 ```Drone<Flying>```，反之亦然
```rust
struct Drone<T> {x: u64, y: u64, z: u64, state: PhantomData<T> }
impl Drone<Idle> {
    fn takeoff(self) -> Drone<Flying> {...}
}
impl Drone<Flying> {
    fn land(self) -> Drone<Idle> { ...}
}
```
    - [▶ 在 Rust Playground 中尝试](https://play.rust-lang.org/)

### Rust 类型状态模式泛型
- 要点：
    - 状态可用结构体表示（零大小）
    - 可将状态 ```T``` 与 ```PhantomData<T>``` 结合（零大小）
    - 为状态机特定阶段实现方法现在只需 ```impl State<T>```
    - 使用消耗 ```self``` 的方法从一个状态转换到另一个状态
    - 这给我们 ```zero cost``` 抽象。编译器可在编译期强制状态机，除非状态正确否则无法调用方法

### Rust 建造者模式 {#rust-builder-pattern}
- 消耗 ```self``` 对建造者模式很有用
- 考虑一个有数十个引脚的 GPIO 配置。引脚可配置为高或低（默认为低）
```rust
#[derive(default)]
enum PinState {
    #[default]
    Low,
    High,
} 
#[derive(default)]
struct GPIOConfig {
    pin0: PinState,
    pin1: PinState
    ... 
}
```
- 建造者模式可用于链式构造 GPIO 配置——[▶ 试试看](https://play.rust-lang.org/)

