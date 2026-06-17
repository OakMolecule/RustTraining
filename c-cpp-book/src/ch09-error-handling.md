## 将枚举与 Option、Result 联系起来 {#connecting-enums-to-option-and-result}

> **你将学到：** Rust 如何用 `Option<T>` 替代空指针、用 `Result<T, E>` 替代异常，以及 `?` 运算符如何简洁传播错误。这是 Rust 最具特色的模式——错误是值，不是隐藏的控制流。

- 还记得前面学的 `enum` 吗？Rust 的 `Option` 与 `Result` 就是标准库中定义的枚举：
```rust
// This is literally how Option is defined in std:
enum Option<T> {
    Some(T),  // Contains a value
    None,     // No value
}

// And Result:
enum Result<T, E> {
    Ok(T),    // Success with value
    Err(E),   // Error with details
}
```
- 因此前面学的 `match` 模式匹配可直接用于 `Option` 与 `Result`
- Rust **没有空指针**——`Option<T>` 是替代，编译器强制处理 `None` 情况

### C++ 对照：异常 vs Result
| **C++ 模式** | **Rust 等价** | **优势** |
|----------------|--------------------|--------------|
| `throw std::runtime_error(msg)` | `Err(MyError::Runtime(msg))` | 错误在返回类型中——不能忘记处理 |
| `try { } catch (...) { }` | `match result { Ok(v) => ..., Err(e) => ... }` | 无隐藏控制流 |
| `std::optional<T>` | `Option<T>` | 须穷尽 match——不能忘记 None |
| `noexcept` 标注 | 默认——所有 Rust 函数都是「noexcept」 | 不存在异常 |
| `errno` / 返回码 | `Result<T, E>` | 类型安全，不能忽略 |

# Rust `Option` 类型 {#rust-option-type}
- Rust ```Option``` 是只有两个变体的 ```enum```：```Some<T>``` 与 ```None```
    - 表示```nullable```类型：要么含该类型的有效值（```Some<T>```），要么无有效值（```None```）
    - ```Option``` 用于操作要么成功返回有效值、要么失败但具体错误无关的 API。例如解析字符串为整数
```rust
fn main() {
    // Returns Option<usize>
    let a = "1234".find("1");
    match a {
        Some(a) => println!("Found 1 at index {a}"),
        None => println!("Couldn't find 1")
    }
}
```

# Rust `Option` 类型
- Rust ```Option``` 有多种处理方式
    - ```unwrap()``` 在 ```Option<T>``` 为 ```None``` 时 panic，否则返回 ```T```，是最不首选的方式
    - ```or()``` 可返回替代值
    - ```if let``` 可测试 ```Some<T>```

> **生产实践：** 参见 [用 unwrap_or 安全取值](ch17-2-avoiding-unchecked-indexing.md#safe-value-extraction-with-unwrap_or) 与 [函数式变换：map、map_err、find_map](ch17-2-avoiding-unchecked-indexing.md#functional-transforms-map-map_err-find_map)，了解生产 Rust 代码中的实例。
```rust
fn main() {
  // This return an Option<usize>
  let a = "1234".find("1");
  println!("{a:?} {}", a.unwrap());
  let a = "1234".find("5").or(Some(42));
  println!("{a:?}");
  if let Some(a) = "1234".find("1") {
      println!("{a}");
  } else {
    println!("Not found in string");
  }
  // This will panic
  // "1234".find("5").unwrap();
}
```

# Rust `Result` 类型 {#rust-result-type}
- Result 是类似 ```Option``` 的 ```enum```，两个变体：```Ok<T>``` 或 ```Err<E>```
    - ```Result``` 广泛用于可能失败的 Rust API。成功时返回 ```Ok<T>```，失败时返回具体错误 ```Err<E>```
```rust
  use std::num::ParseIntError;
  fn main() {
  let a : Result<i32, ParseIntError>  = "1234z".parse();
  match a {
      Ok(n) => println!("Parsed {n}"),
      Err(e) => println!("Parsing failed {e:?}"),
  }
  let a : Result<i32, ParseIntError>  = "1234z".parse().or(Ok(-1));
  println!("{a:?}");
  if let Ok(a) = "1234".parse::<i32>() {
    println!("Let OK {a}");  
  }
  // This will panic
  //"1234z".parse().unwrap();
}
```

## Option 与 Result：同一枚硬币的两面

`Option` 与 `Result` 密切相关——`Option<T>` 本质上是 `Result<T, ()>`（错误不带信息的结果）：

| `Option<T>` | `Result<T, E>` | 含义 |
|-------------|---------------|---------|
| `Some(value)` | `Ok(value)` | 成功——有值 |
| `None` | `Err(error)` | 失败——无值（Option）或错误详情（Result） |

**相互转换：**

```rust
fn main() {
    let opt: Option<i32> = Some(42);
    let res: Result<i32, &str> = opt.ok_or("value was None");  // Option → Result
    
    let res: Result<i32, &str> = Ok(42);
    let opt: Option<i32> = res.ok();  // Result → Option (discards error)
    
    // They share many of the same methods:
    // .map(), .and_then(), .unwrap_or(), .unwrap_or_else(), .is_some()/is_ok()
}
```

> **经验法则**：缺失是正常情况时用 `Option`（如查键）。失败需要解释时用 `Result`（如文件 I/O、解析）。

# 练习：用 Option 实现 log() 函数 {#exercise-log-function-implementation-with-option}

🟢 **入门**

- 实现 ```log()```，接受 ```Option<&str>``` 参数。参数为 ```None``` 时打印默认字符串
- 函数返回 ```Result```，成功与错误类型均为 ```()```（本例永不产生错误）

<details><summary>Solution (click to expand)</summary>

```rust
fn log(message: Option<&str>) -> Result<(), ()> {
    match message {
        Some(msg) => println!("LOG: {msg}"),
        None => println!("LOG: (no message provided)"),
    }
    Ok(())
}

fn main() {
    let _ = log(Some("System initialized"));
    let _ = log(None);
    
    // Alternative using unwrap_or:
    let msg: Option<&str> = None;
    println!("LOG: {}", msg.unwrap_or("(default message)"));
}
// Output:
// LOG: System initialized
// LOG: (no message provided)
// LOG: (default message)
```

</details>

----
# Rust 错误处理 {#rust-error-handling}
 - Rust 错误可分为不可恢复（致命）与可恢复。致命错误导致 ``panic``
    - 一般应避免导致 ```panic``` 的情况。```panic``` 由程序 bug 引起，包括越界索引、对 ```Option<None>``` 调用 ```unwrap()``` 等
    - 对理论上不可能的条件显式 ```panic``` 是可以的。可用 ```panic!``` 或 ```assert!``` 做健全性检查
```rust
fn main() {
   let x : Option<u32> = None;
   // println!("{x}", x.unwrap()); // Will panic
   println!("{}", x.unwrap_or(0));  // OK -- prints 0
   let x = 41;
   //assert!(x == 42); // Will panic
   //panic!("Something went wrong"); // Unconditional panic
   let _a = vec![0, 1];
   // println!("{}", a[2]); // Out of bounds panic; use a.get(2) which will return Option<T>
}
```

## 错误处理：C++ vs Rust

### C++ 基于异常的错误处理问题

```cpp
// C++ error handling - exceptions create hidden control flow
#include <fstream>
#include <stdexcept>

std::string read_config(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open()) {
        throw std::runtime_error("Cannot open: " + path);
    }
    std::string content;
    // What if getline throws? Is file properly closed?
    // With RAII yes, but what about other resources?
    std::getline(file, content);
    return content;  // What if caller doesn't try/catch?
}

int main() {
    // ERROR: Forgot to wrap in try/catch!
    auto config = read_config("nonexistent.txt");
    // Exception propagates silently, program crashes
    // Nothing in the function signature warned us
    return 0;
}
```

```mermaid
graph TD
    subgraph "C++ Error Handling Issues"
        CF["Function Call"]
        CR["throw exception<br/>or return code"]
        CIGNORE["[ERROR] Exception not caught<br/>or return code ignored"]
        CCHECK["try/catch or check"]
        CERROR["Hidden control flow<br/>throws not in signature"]
        CERRNO["No compile-time<br/>enforcement"]
        
        CF --> CR
        CR --> CIGNORE
        CR --> CCHECK
        CCHECK --> CERROR
        CERROR --> CERRNO
        
        CPROBLEMS["[ERROR] Exceptions invisible in types<br/>[ERROR] Hidden control flow<br/>[ERROR] Easy to forget try/catch<br/>[ERROR] Exception safety is hard<br/>[ERROR] noexcept is opt-in"]
    end
    
    subgraph "Rust Result<T, E> System"
        RF["Function Call"]
        RR["Result<T, E><br/>Ok(value) | Err(error)"]
        RMUST["[OK] Must handle<br/>Compile error if ignored"]
        RMATCH["Pattern matching<br/>match, if let, ?"]
        RDETAIL["Detailed error info<br/>Custom error types"]
        RSAFE["Type-safe<br/>No global state"]
        
        RF --> RR
        RR --> RMUST
        RMUST --> RMATCH
        RMATCH --> RDETAIL
        RDETAIL --> RSAFE
        
        RBENEFITS["[OK] Forced error handling<br/>[OK] Type-safe errors<br/>[OK] Detailed error info<br/>[OK] Composable with ?<br/>[OK] Zero runtime cost"]
    end
    
    style CPROBLEMS fill:#ff6b6b,color:#000
    style RBENEFITS fill:#91e5a3,color:#000
    style CIGNORE fill:#ff6b6b,color:#000
    style RMUST fill:#91e5a3,color:#000
```

### `Result<T, E>` 可视化

```rust
// Rust error handling - comprehensive and forced
use std::fs::File;
use std::io::Read;

fn read_file_content(filename: &str) -> Result<String, std::io::Error> {
    let mut file = File::open(filename)?;  // ? automatically propagates errors
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)  // Success case
}

fn main() {
    match read_file_content("example.txt") {
        Ok(content) => println!("File content: {}", content),
        Err(error) => println!("Failed to read file: {}", error),
        // Compiler forces us to handle both cases!
    }
}
```

```mermaid
graph TD
    subgraph "Result<T, E> Flow"
        START["Function starts"]
        OP1["File::open()"]
        CHECK1{{"Result check"}}
        OP2["file.read_to_string()"]
        CHECK2{{"Result check"}}
        SUCCESS["Ok(contents)"]
        ERROR1["Err(io::Error)"]
        ERROR2["Err(io::Error)"]
        
        START --> OP1
        OP1 --> CHECK1
        CHECK1 -->|"Ok(file)"| OP2
        CHECK1 -->|"Err(e)"| ERROR1
        OP2 --> CHECK2
        CHECK2 -->|"Ok(())"| SUCCESS
        CHECK2 -->|"Err(e)"| ERROR2
        
        ERROR1 --> PROPAGATE["? operator<br/>propagates error"]
        ERROR2 --> PROPAGATE
        PROPAGATE --> CALLER["Caller must<br/>handle error"]
    end
    
    subgraph "Pattern Matching Options"
        MATCH["match result"]
        IFLET["if let Ok(val) = result"]
        UNWRAP["result.unwrap()<br/>[WARNING] Panics on error"]
        EXPECT["result.expect(msg)<br/>[WARNING] Panics with message"]
        UNWRAP_OR["result.unwrap_or(default)<br/>[OK] Safe fallback"]
        QUESTION["result?<br/>[OK] Early return"]
        
        MATCH --> SAFE1["[OK] Handles both cases"]
        IFLET --> SAFE2["[OK] Handles error case"]
        UNWRAP_OR --> SAFE3["[OK] Always returns value"]
        QUESTION --> SAFE4["[OK] Propagates to caller"]
        UNWRAP --> UNSAFE1["[ERROR] Can panic"]
        EXPECT --> UNSAFE2["[ERROR] Can panic"]
    end
    
    style SUCCESS fill:#91e5a3,color:#000
    style ERROR1 fill:#ffa07a,color:#000
    style ERROR2 fill:#ffa07a,color:#000
    style SAFE1 fill:#91e5a3,color:#000
    style SAFE2 fill:#91e5a3,color:#000
    style SAFE3 fill:#91e5a3,color:#000
    style SAFE4 fill:#91e5a3,color:#000
    style UNSAFE1 fill:#ff6b6b,color:#000
    style UNSAFE2 fill:#ff6b6b,color:#000
```

# Rust 错误处理
- Rust 用 ```enum Result<T, E>``` 处理可恢复错误
    - ```Ok<T>``` 变体含成功结果，```Err<E>``` 含错误
```rust
fn main() {
    let x = "1234x".parse::<u32>();
    match x {
        Ok(x) => println!("Parsed number {x}"),
        Err(e) => println!("Parsing error {e:?}"),
    }
    let x  = "1234".parse::<u32>();
    // Same as above, but with valid number
    if let Ok(x) = &x {
        println!("Parsed number {x}")
    } else if let Err(e) = &x {
        println!("Error: {e:?}");
    }
}
```

# Rust 错误处理
- try 运算符 ```?``` 是 ```match``` ```Ok``` / ```Err``` 模式的便捷简写
    - 方法须返回 ```Result<T, E>``` 才能使用 ```?```
    - ```Result<T, E>``` 的类型可改。下例返回与 ```str::parse()``` 相同的错误类型（```std::num::ParseIntError```）
```rust
fn double_string_number(s : &str) -> Result<u32, std::num::ParseIntError> {
   let x = s.parse::<u32>()?; // Returns immediately in case of an error
   Ok(x*2)
}
fn main() {
    let result = double_string_number("1234");
    println!("{result:?}");
    let result = double_string_number("1234x");
    println!("{result:?}");
}
```

# Rust 错误处理
- 错误可映射为其他类型或默认值（https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_or_default）
```rust
// Changes the error type to () in case of error
fn double_string_number(s : &str) -> Result<u32, ()> {
   let x = s.parse::<u32>().map_err(|_|())?; // Returns immediately in case of an error
   Ok(x*2)
}
```
```rust
fn double_string_number(s : &str) -> Result<u32, ()> {
   let x = s.parse::<u32>().unwrap_or_default(); // Defaults to 0 in case of parse error
   Ok(x*2)
}
```
```rust
fn double_optional_number(x : Option<u32>) -> Result<u32, ()> {
    // ok_or converts Option<None> to Result<u32, ()> in the below
    x.ok_or(()).map(|x|x*2) // .map() is applied only on Ok(u32)
}
```

# 练习：错误处理 {#exercise-error-handling}

🟡 **中级**
- 实现带单个 u32 参数的 ```log()```。参数不是 42 则返回错误。成功与错误的 ```Result<>``` 类型均为 ```()```
- 调用 ```log()``` 的函数在 ```log()``` 返回错误时立即退出，类型同为 ```Result<>```；否则打印 log 调用成功

```rust
fn log(x: u32) -> ?? {

}

fn call_log(x: u32) -> ?? {
    // Call log(x), then exit immediately if it return an error
    println!("log was successfully called");
}

fn main() {
    call_log(42);
    call_log(43);
}
``` 

<details><summary>Solution (click to expand)</summary>

```rust
fn log(x: u32) -> Result<(), ()> {
    if x == 42 {
        Ok(())
    } else {
        Err(())
    }
}

fn call_log(x: u32) -> Result<(), ()> {
    log(x)?;  // Exit immediately if log() returns an error
    println!("log was successfully called with {x}");
    Ok(())
}

fn main() {
    let _ = call_log(42);  // Prints: log was successfully called with 42
    let _ = call_log(43);  // Returns Err(()), nothing printed
}
// Output:
// log was successfully called with 42
```

</details>


