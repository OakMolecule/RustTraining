# Rust `if` 关键字 {#rust-if-keyword}

> **你将学到：** Rust 的控制流结构——作为表达式的 `if`/`else`、`loop`/`while`/`for`、`match`，以及它们与 C/C++ 对应物的差异。关键洞察：大多数 Rust 控制流会返回值。

- 在 Rust 中，```if``` 实际上是表达式，即可用于赋值，也表现得像语句。[▶ 试一试](https://play.rust-lang.org/)

```rust
fn main() {
    let x = 42;
    if x < 42 {
        println!("Smaller than the secret of life");
    } else if x == 42 {
        println!("Is equal to the secret of life");
    } else {
        println!("Larger than the secret of life");
    }
    let is_secret_of_life = if x == 42 {true} else {false};
    println!("{}", is_secret_of_life);
}
```

# Rust 使用 `while` 和 `for` 的循环 {#rust-loops-using-while-and-for}
- ```while``` 关键字可在表达式为真时循环
```rust
fn main() {
    let mut x = 40;
    while x != 42 {
        x += 1;
    }
}
```
- ```for``` 关键字可遍历范围
```rust
fn main() {
    // Will not print 43; use 40..=43 to include last element
    for x in 40..43 {
        println!("{}", x);
    } 
}
```

# Rust 使用 `loop` 的循环 {#rust-loops-using-loop}
- ```loop``` 关键字创建无限循环，直到遇到 ```break```
```rust
fn main() {
    let mut x = 40;
    // Change the below to 'here: loop to specify optional label for the loop
    loop {
        if x == 42 {
            break; // Use break x; to return the value of x
        }
        x += 1;
    }
}
```
- ```break``` 语句可包含可选表达式，用于为 ```loop``` 表达式赋值
- ```continue``` 关键字可返回 ```loop``` 顶部
- 循环标签可与 ```break``` 或 ```continue``` 配合使用，在处理嵌套循环时很有用

# Rust 表达式块 {#rust-expression-blocks}
- Rust 表达式块是放在 ```{}``` 中的一串表达式。求值结果为块中最后一个表达式
```rust
fn main() {
    let x = {
        let y = 40;
        y + 2 // Note: ; must be omitted
    };
    // Notice the Python style printing
    println!("{x}");
}
```
- Rust 风格是用此省略函数中的 ```return``` 关键字
```rust
fn is_secret_of_life(x: u32) -> bool {
    // Same as if x == 42 {true} else {false}
    x == 42 // Note: ; must be omitted 
}
fn main() {
    println!("{}", is_secret_of_life(42));
}
```

