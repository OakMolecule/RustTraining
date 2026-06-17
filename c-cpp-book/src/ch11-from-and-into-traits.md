# Rust From 与 Into Trait {#rust-from-and-into-traits}

> **你将学到：** Rust 的类型转换 Trait——用于不会失败的转换的 `From<T>` 和 `Into<T>`，用于可能失败的 `TryFrom` 和 `TryInto`。实现 `From` 即可免费获得 `Into`。替代 C++ 转换运算符和构造函数。

- ```From``` 和 ```Into``` 是便于类型转换的互补 Trait
- 类型通常实现 ```From``` Trait。```String::from("Rust")``` 将 ```&str``` 转换为 ```String```。
当存在 ```From<T> for U``` 时，Rust 还提供 ```Into<U> for T```，因此 ```let s: String = "Rust".into();``` 也能工作。
```rust
struct Point {x: u32, y: u32}
// Construct a Point from a tuple
impl From<(u32, u32)> for Point {
    fn from(xy : (u32, u32)) -> Self {
        Point {x : xy.0, y: xy.1}       // Construct Point using the tuple elements
    }
}
fn main() {
    let s = String::from("Rust");
    let x = u32::from(true);
    let p = Point::from((40, 42));
    // let p : Point = (40,42).into(); // Alternate form of the above
    println!("s: {s} x:{x} p.x:{} p.y {}", p.x, p.y);   
}
```

# 练习：From 与 Into {#exercise-from-and-into}
- 为 ```Point``` 实现 ```From``` Trait，转换为名为 ```TransposePoint``` 的类型。```TransposePoint``` 交换 ```Point``` 的 ```x``` 和 ```y``` 元素

<details><summary>Solution (click to expand)</summary>

```rust
struct Point { x: u32, y: u32 }
struct TransposePoint { x: u32, y: u32 }

impl From<Point> for TransposePoint {
    fn from(p: Point) -> Self {
        TransposePoint { x: p.y, y: p.x }
    }
}

fn main() {
    let p = Point { x: 10, y: 20 };
    let tp = TransposePoint::from(p);
    println!("Transposed: x={}, y={}", tp.x, tp.y);  // x=20, y=10

    // Using .into() — works automatically when From is implemented
    let p2 = Point { x: 3, y: 7 };
    let tp2: TransposePoint = p2.into();
    println!("Transposed: x={}, y={}", tp2.x, tp2.y);  // x=7, y=3
}
// Output:
// Transposed: x=20, y=10
// Transposed: x=7, y=3
```

</details>

# Rust Default Trait {#rust-default-trait}
- ```Default``` 可用于为类型实现默认值
    - 类型可使用 ```Derive``` 宏配合 ```Default```，或提供自定义实现
```rust
#[derive(Default, Debug)]
struct Point {x: u32, y: u32}
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = Point::default();   // Creates a Point{0, 0}
    println!("{x:?}");
    let y = CustomPoint::default();
    println!("{y:?}");
}
```

### Rust Default Trait
- ```Default``` Trait 有多种用途，包括
    - 执行部分复制，其余用默认初始化
    - 作为 ```Option``` 类型在 ```unwrap_or_default()``` 等方法中的默认替代
```rust
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = CustomPoint::default();
    // Override y, but leave rest of elements as the default
    let y = CustomPoint {y: 43, ..CustomPoint::default()};
    println!("{x:?} {y:?}");
    let z : Option<CustomPoint> = None;
    // Try changing the unwrap_or_default() to unwrap()
    println!("{:?}", z.unwrap_or_default());
}
```

### 其他 Rust 类型转换 {#other-rust-type-conversions}
- Rust 不支持隐式类型转换，```as``` 可用于```显式```转换
- ```as``` 应谨慎使用，因为它可能因窄化等导致数据丢失。一般而言，尽可能使用 ```into()``` 或 ```from()```
```rust
fn main() {
    let f = 42u8;
    // let g : u32 = f;    // Will not compile
    let g = f as u32;      // Ok, but not preferred. Subject to rules around narrowing
    let g : u32 = f.into(); // Most preferred form; infallible and checked by the compiler
    // let k : u8 = g.into();  // Fails to compile; narrowing can result in loss of data
    
    // Attempting a narrowing operation requires use of try_into
    if let Ok(k) = TryInto::<u8>::try_into(g) {
        println!("{k}");
    }
}
```

