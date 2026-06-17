# Rust 内置类型 {#built-in-rust-types}

> **你将学到：** Rust 的基本类型（`i32`、`u64`、`f64`、`bool`、`char`）、类型推断、显式类型标注，以及它们与 C/C++ 基本类型的对比。无隐式转换——Rust 要求显式类型转换。

- Rust 支持类型推断，也允许显式指定类型

|  **说明**  |            **类型**            |          **示例**          |
|:-----------------:|:------------------------------:|:-----------------------------:|
| 有符号整数   | i8, i16, i32, i64, i128, isize | -1, 42, 1_00_000, 1_00_000i64 |
| 无符号整数 | u8, u16, u32, u64, u128, usize | 0, 42, 42u32, 42u64           |
| 浮点数    | f32, f64                       | 0.0, 0.42                     |
| Unicode           | char                           | 'a', '$'                      |
| 布尔值           | bool                           | true, false                   |

- Rust 允许在数字中任意使用 ```_``` 以提高可读性
----
### Rust 类型指定与赋值 {#rust-type-specification-and-assignment}
- Rust 使用 ```let``` 关键字为变量赋值。变量类型可在 ```:``` 后可选指定
```rust
fn main() {
    let x : i32 = 42;
    // These two assignments are logically equivalent
    let y : u32 = 42;
    let z = 42u32;
}
``` 
- 函数参数和返回值（如有）需要显式类型。以下函数接受 u8 参数并返回 u32
```rust
fn foo(x : u8) -> u32
{
    return x as u32 * x as u32;
}
```
- 未使用的变量以 ```_``` 为前缀以避免编译器警告
----
# Rust 类型指定与推断 {#rust-type-specification-and-inference}
- Rust 可根据上下文自动推断变量类型。
- [▶ 在 Rust Playground 中尝试](https://play.rust-lang.org/)
```rust
fn secret_of_life_u32(x : u32) {
    println!("The u32 secret_of_life is {}", x);
}

fn secret_of_life_u8(x : u8) {
    println!("The u8 secret_of_life is {}", x);
}

fn main() {
    let a = 42; // The let keyword assigns a value; type of a is u32
    let b = 42; // The let keyword assigns a value; inferred type of b is u8
    secret_of_life_u32(a);
    secret_of_life_u8(b);
}
```

# Rust 变量与可变性 {#rust-variables-and-mutability}
- Rust 变量**默认不可变**，除非使用 ```mut``` 关键字声明可变。例如，以下代码除非将 ```let a = 42``` 改为 ```let mut a = 42```，否则无法编译
```rust
fn main() {
    let a = 42; // Must be changed to let mut a = 42 to permit the assignment below 
    a = 43;  // Will not compile unless the above is changed
}
```
- Rust 允许重用变量名（遮蔽，shadowing）
```rust
fn main() {
    let a = 42;
    {
        let a = 43; //OK: Different variable with the same name
    }
    // a = 43; // Not permitted
    let a = 43; // Ok: New variable and assignment
}
```


