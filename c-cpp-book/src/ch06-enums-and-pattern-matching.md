# Rust 枚举类型 {#rust-enum-types}

> **你将学到：** Rust 枚举作为可辨识联合体（tagged union 的正确做法）、用于穷尽模式匹配的 `match`，以及枚举如何以编译器强制安全的方式替代 C++ 类层次与 C 的 tagged union。

- 枚举类型是可辨识联合体，即若干可能类型的和类型，带标识具体变体的标签
    - 面向 C 开发者：Rust 枚举可携带数据（tagged union 的正确做法——编译器跟踪当前活跃变体）
    - 面向 C++ 开发者：Rust 枚举类似 `std::variant`，但有穷尽模式匹配、无 `std::get` 异常、无 `std::visit` 样板代码
    - `enum` 的大小等于最大可能类型的大小。各变体彼此无关，可有完全不同的类型
    - `enum` 是语言最强大特性之一——在 C++ 中可替代整棵类层次（案例研究中有更多说明）
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let a = Numbers::Zero;
    let b = Numbers::SmallNumber(42);
    let c : Numbers = a; // Ok -- the type of a is Numbers
    let d : Numbers = b; // Ok -- the type of b is Numbers
}
```
----
# Rust `match` 语句 {#rust-match-statement}
- Rust 的 ```match``` 相当于加强版 C 的 `switch`
    - ```match``` 可用于简单数据类型、```struct```、```enum``` 的模式匹配
    - ```match``` 必须穷尽，即覆盖给定 ```type``` 的所有可能情况。```_``` 可作「其余所有」的通配符
    - ```match``` 可产生值，但各分支（```=>```）必须返回相同类型的值

```rust
fn main() {
    let x = 42;
    // In this case, the _ covers all numbers except the ones explicitly listed
    let is_secret_of_life = match x {
        42 => true, // return type is boolean value
        _ => false, // return type boolean value
        // This won't compile because return type isn't boolean
        // _ => 0  
    };
    println!("{is_secret_of_life}");
}
```

# Rust `match` 语句
- ```match``` 支持范围、布尔过滤与 ```if``` 守卫
```rust
fn main() {
    let x = 42;
    match x {
        // Note that the =41 ensures the inclusive range
        0..=41 => println!("Less than the secret of life"),
        42 => println!("Secret of life"),
        _ => println!("More than the secret of life"),
    }
    let y = 100;
    match y {
        100 if x == 43 => println!("y is 100% not secret of life"),
        100 if x == 42 => println!("y is 100% secret of life"),
        _ => (),    // Do nothing
    }
}
```

# Rust `match` 语句
- ```match``` 常与 ```enum``` 结合
    - match 可将内含值绑定到变量。若值不关心，用 ```_```
    - ```matches!``` 宏可匹配特定变体
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let b = Numbers::SmallNumber(42);
    match b {
        Numbers::Zero => println!("Zero"),
        Numbers::SmallNumber(value) => println!("Small number {value}"),
        Numbers::BiggerNumber(_) | Numbers::EvenBiggerNumber(_) => println!("Some BiggerNumber or EvenBiggerNumber"),
    }
    
    // Boolean test for specific variants
    if matches!(b, Numbers::Zero | Numbers::SmallNumber(_)) {
        println!("Matched Zero or small number");
    }
}
```

# Rust `match` 语句
- ```match``` 也可通过解构与切片匹配
```rust
fn main() {
    struct Foo {
        x: (u32, bool),
        y: u32
    }
    let f = Foo {x: (42, true), y: 100};
    match f {
        // Capture the value of x into a variable called tuple
        Foo{y: 100, x : tuple} => println!("Matched x: {tuple:?}"),
        _ => ()
    }
    let a = [40, 41, 42];
    match a {
        // Last element of slice must be 42. @ is used to bind the match
        [rest @ .., 42] => println!("{rest:?}"),
        // First element of the slice must be 42. @ is used to bind the match
        [42, rest @ ..] => println!("{rest:?}"),
        _ => (),
    }
}
```

# 练习：用 `match` 与 `enum` 实现加减 {#exercise-implement-add-and-subtract-using-match-and-enum}

🟢 **入门**

- 编写函数，对无符号 64 位整数实现算术运算
- **步骤 1**：定义运算枚举：
```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}
```
- **步骤 2**：定义结果枚举：
```rust
enum CalcResult {
    Ok(u64),                    // Successful result
    Invalid(String),            // Error message for invalid operations
}
```
- **步骤 3**：实现 `calculate(op: Operation) -> CalcResult`
    - Add：返回 Ok(和)
    - Subtract：若第一个 >= 第二个则返回 Ok(差)，否则 Invalid("Underflow")
- **提示**：在函数中使用模式匹配：
```rust
match op {
    Operation::Add(a, b) => { /* your code */ },
    Operation::Subtract(a, b) => { /* your code */ },
}
```

<details><summary>Solution (click to expand)</summary>

```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}

enum CalcResult {
    Ok(u64),
    Invalid(String),
}

fn calculate(op: Operation) -> CalcResult {
    match op {
        Operation::Add(a, b) => CalcResult::Ok(a + b),
        Operation::Subtract(a, b) => {
            if a >= b {
                CalcResult::Ok(a - b)
            } else {
                CalcResult::Invalid("Underflow".to_string())
            }
        }
    }
}

fn main() {
    match calculate(Operation::Add(10, 20)) {
        CalcResult::Ok(result) => println!("10 + 20 = {result}"),
        CalcResult::Invalid(msg) => println!("Error: {msg}"),
    }
    match calculate(Operation::Subtract(5, 10)) {
        CalcResult::Ok(result) => println!("5 - 10 = {result}"),
        CalcResult::Invalid(msg) => println!("Error: {msg}"),
    }
}
// Output:
// 10 + 20 = 30
// Error: Underflow
```

</details>

# Rust 关联方法
- ```impl``` 可为 ```struct```、```enum``` 等类型定义关联方法
    - 方法可选地接受 ```self``` 参数。```self``` 概念上类似 C 中把结构体指针作为首参，或 C++ 中的 ```this```
    - 对 ```self``` 的引用可为不可变（默认 ```&self```）、可变（```&mut self```）或 ```self```（转移所有权）
    - ```Self``` 关键字可作类型简写
```rust
struct Point {x: u32, y: u32}
impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point {x, y}
    }
    fn increment_x(&mut self) {
        self.x += 1;
    }
}
fn main() {
    let mut p = Point::new(10, 20);
    p.increment_x();
}
```

# 练习：`Point` 的 add 与 transform

🟡 **中级** — 需理解方法签名中的移动 vs 借用
- 为 ```Point``` 实现下列关联方法
    - ```add()``` 接收另一个 ```Point```，原地增加 x、y（提示：使用 ```&mut self```）
    - ```transform()``` 消费现有 ```Point```（提示：使用 ```self```），返回 x、y 平方后的新 ```Point```

<details><summary>Solution (click to expand)</summary>

```rust
struct Point { x: u32, y: u32 }

impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point { x, y }
    }
    fn add(&mut self, other: &Point) {
        self.x += other.x;
        self.y += other.y;
    }
    fn transform(self) -> Point {
        Point { x: self.x * self.x, y: self.y * self.y }
    }
}

fn main() {
    let mut p1 = Point::new(2, 3);
    let p2 = Point::new(10, 20);
    p1.add(&p2);
    println!("After add: x={}, y={}", p1.x, p1.y);           // x=12, y=23
    let p3 = p1.transform();
    println!("After transform: x={}, y={}", p3.x, p3.y);     // x=144, y=529
    // p1 is no longer accessible — transform() consumed it
}
```

</details>

----

