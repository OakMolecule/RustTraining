### Unsafe Rust {#unsafe-rust}

> **你将学到：** 何时及如何使用 `unsafe`——裸指针解引用、用于从 Rust 调用 C 及反向调用的 FFI（Foreign Function Interface）、用于字符串互操作的 `CString`/`CStr`，以及如何围绕 unsafe 代码编写 Safe Rust 包装。

- ```unsafe``` 解锁通常被 Rust 编译器禁止的功能
    - 解引用裸指针
    - 访问*可变*静态变量
    - https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
- 能力越大，责任越大
    - ```unsafe``` 告诉编译器「我，程序员，承担维护编译器通常保证的不变式的责任」
    - 必须保证没有别名化的可变与不可变引用、没有悬垂指针、没有无效引用，……
    - ```unsafe``` 的使用应限制在尽可能小的作用域
    - 所有使用 ```unsafe``` 的代码都应有描述假设的「safety」注释

### Unsafe Rust 示例
```rust
unsafe fn harmless() {}
fn main() {
    // Safety: We are calling a harmless unsafe function
    unsafe {
        harmless();
    }
    let a = 42u32;
    let p = &a as *const u32;
    // Safety: p is a valid pointer to a variable that will remain in scope
    unsafe {
        println!("{}", *p);
    }
    // Safety: Not safe; for illustration purposes only
    let dangerous_buffer = 0xb8000 as *mut u32;
    unsafe {
        println!("About to go kaboom!!!");
        *dangerous_buffer = 0; // This will SEGV on most modern machines
    }
}
```

### 简单 FFI 示例（C 消费的 Rust 库函数） {#simple-ffi-example-rust-library-function-consumed-by-c}

## FFI 字符串：CString 与 CStr

FFI 表示 *Foreign Function Interface*（外部函数接口）——Rust 用于调用其他语言（如 C）编写的函数及反向调用的机制。

与 C 代码交互时，Rust 的 `String` 和 `&str` 类型（UTF-8 且无 null 终止符）与 C 字符串（null 终止的字节数组）不直接兼容。Rust 在 `std::ffi` 中提供 `CString`（拥有）和 `CStr`（借用）：

| 类型 | 类比 | 何时使用 |
|------|-------------|----------|
| `CString` | `String`（拥有） | 从 Rust 数据创建 C 字符串 |
| `&CStr` | `&str`（借用） | 从外部代码接收 C 字符串 |

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

fn demo_ffi_strings() {
    // Creating a C-compatible string (adds null terminator)
    let c_string = CString::new("Hello from Rust").expect("CString::new failed");
    let ptr: *const c_char = c_string.as_ptr();

    // Converting a C string back to Rust (unsafe because we trust the pointer)
    // Safety: ptr is valid and null-terminated (we just created it above)
    let back_to_rust: &CStr = unsafe { CStr::from_ptr(ptr) };
    let rust_str: &str = back_to_rust.to_str().expect("Invalid UTF-8");
    println!("{}", rust_str);
}
```

> **警告**：若输入包含内部 null 字节（`\0`），`CString::new()` 会返回错误。始终处理 `Result`。下文 FFI 示例中会大量使用 `CStr`。

- ```FFI``` 方法必须用 ```#[no_mangle]``` 标记，以确保编译器不修改（mangle）函数名
- 我们将 crate 编译为静态库
    ```
    #[no_mangle] 
    pub extern "C" fn add(left: u64, right: u64) -> u64 {
        left + right
    }
    ```
- 我们将编译以下 C 代码并链接到静态库。
    ```
    #include <stdio.h>
    #include <stdint.h>
    extern uint64_t add(uint64_t, uint64_t);
    int main() {
        printf("Add returned %llu\n", add(21, 21));
    }
    ``` 

### 复杂 FFI 示例 {#complex-ffi-example}
- 在以下示例中，我们将创建 Rust 日志接口并暴露给
[PYTHON] 和 ```C```
    - 我们将看到同一接口如何原生用于 Rust 和 C
    - 我们将探索使用 ```cbindgen``` 等工具为 ```C``` 生成头文件
    - 我们将看到 ```unsafe``` 包装如何作为通往 Safe Rust 代码的桥梁

## Logger 辅助函数
```rust
fn create_or_open_log_file(log_file: &str, overwrite: bool) -> Result<File, String> {
    if overwrite {
        File::create(log_file).map_err(|e| e.to_string())
    } else {
        OpenOptions::new()
            .write(true)
            .append(true)
            .open(log_file)
            .map_err(|e| e.to_string())
    }
}

fn log_to_file(file_handle: &mut File, message: &str) -> Result<(), String> {
    file_handle
        .write_all(message.as_bytes())
        .map_err(|e| e.to_string())
}
```

## Logger 结构体
```rust
struct SimpleLogger {
    log_level: LogLevel,
    file_handle: File,
}

impl SimpleLogger {
    fn new(log_file: &str, overwrite: bool, log_level: LogLevel) -> Result<Self, String> {
        let file_handle = create_or_open_log_file(log_file, overwrite)?;
        Ok(Self {
            file_handle,
            log_level,
        })
    }

    fn log_message(&mut self, log_level: LogLevel, message: &str) -> Result<(), String> {
        if log_level as u32 <= self.log_level as u32 {
            let timestamp = Local::now().format("%Y-%m-%d %H:%M:%S").to_string();
            let message = format!("Simple: {timestamp} {log_level} {message}\n");
            log_to_file(&mut self.file_handle, &message)
        } else {
            Ok(())
        }
    }
}
```

## 测试
- 用 Rust 测试功能很简单
    - 测试方法用 ```#[test]``` 装饰，不是编译二进制的一部分
    - 很容易为测试目的创建 mock 方法
```rust
#[test]
fn testfunc() -> Result<(), String> {
    let mut logger = SimpleLogger::new("test.log", false, LogLevel::INFO)?;
    logger.log_message(LogLevel::TRACELEVEL1, "Hello world")?;
    logger.log_message(LogLevel::CRITICAL, "Critical message")?;
    Ok(()) // The compiler automatically drops logger here
}
```
```bash
cargo test
```

## (C)-Rust FFI
- cbindgen 是生成导出 Rust 函数头文件的优秀工具
    - 可用 cargo 安装
```bash
cargo install cbindgen
cbindgen 
```
- 函数和结构体可使用 ```#[no_mangle]``` 和 ```#[repr(C)]``` 导出
    - 我们假设常见的接口模式：传入指向实际实现的 `**`，成功返回 0，失败返回非零
    - **不透明与透明结构体**：我们的 `SimpleLogger` 作为*不透明指针*（`*mut SimpleLogger`）传递——C 端从不访问其字段，因此**不需要** `#[repr(C)]`。当 C 代码需要直接读/写结构体字段时使用 `#[repr(C)]`：

```rust
// Opaque — C only holds a pointer, never inspects fields. No #[repr(C)] needed.
struct SimpleLogger { /* Rust-only fields */ }

// Transparent — C reads/writes fields directly. MUST use #[repr(C)].
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
```
```c
typedef struct SimpleLogger SimpleLogger;
uint32_t create_simple_logger(const char *file_name, struct SimpleLogger **out_logger);
uint32_t log_entry(struct SimpleLogger *logger, const char *message);
uint32_t drop_logger(struct SimpleLogger *logger);
```

- 注意我们需要大量健全性检查
- 必须显式泄漏内存以防止 Rust 自动释放
```rust
#[no_mangle] 
pub extern "C" fn create_simple_logger(file_name: *const std::os::raw::c_char, out_logger: *mut *mut SimpleLogger) -> u32 {
    use std::ffi::CStr;
    // Make sure pointer isn't NULL
    if file_name.is_null() || out_logger.is_null() {
        return 1;
    }
    // Safety: The passed in pointer is either NULL or 0-terminated by contract
    let file_name = unsafe {
        CStr::from_ptr(file_name)
    };
    let file_name = file_name.to_str();
    // Make sure that file_name doesn't have garbage characters
    if file_name.is_err() {
        return 1;
    }
    let file_name = file_name.unwrap();
    // Assume some defaults; we'll pass them in real life
    let new_logger = SimpleLogger::new(file_name, true, LogLevel::CRITICAL);
    // Check that we were able to construct the logger
    if new_logger.is_err() {
        return 1;
    }
    let new_logger = Box::new(new_logger.unwrap());
    // This prevents the Box from being dropped when if goes out of scope
    let logger_ptr: *mut SimpleLogger = Box::leak(new_logger);
    // Safety: logger is non-null and logger_ptr is valid
    unsafe {
        *out_logger = logger_ptr;
    }
    return 0;
}
```

- ```log_entry()``` 中有类似的错误检查
```rust
#[no_mangle]
pub extern "C" fn log_entry(logger: *mut SimpleLogger, message: *const std::os::raw::c_char) -> u32 {
    use std::ffi::CStr;
    if message.is_null() || logger.is_null() {
        return 1;
    }
    // Safety: message is non-null
    let message = unsafe {
        CStr::from_ptr(message)
    };
    let message = message.to_str();
    // Make sure that file_name doesn't have garbage characters
    if message.is_err() {
        return 1;
    }
    // Safety: logger is valid pointer previously constructed by create_simple_logger()
    unsafe {
        (*logger).log_message(LogLevel::CRITICAL, message.unwrap()).is_err() as u32
    }
}

#[no_mangle]
pub extern "C" fn drop_logger(logger: *mut SimpleLogger) -> u32 {
    if logger.is_null() {
        return 1;
    }
    // Safety: logger is valid pointer previously constructed by create_simple_logger()
    unsafe {
        // This constructs a Box<SimpleLogger>, which is dropped when it goes out of scope
        let _ = Box::from_raw(logger);
    }
    0
}
```

- 可用 Rust 或编写 (C) 程序测试我们的 (C)-FFI
```rust
#[test]
fn test_c_logger() {
    // The c".." creates a NULL terminated string
    let file_name = c"test.log".as_ptr() as *const std::os::raw::c_char;
    let mut c_logger: *mut SimpleLogger = std::ptr::null_mut();
    assert_eq!(create_simple_logger(file_name, &mut c_logger), 0);
    // This is the manual way to create c"..." strings
    let message = b"message from C\0".as_ptr() as *const std::os::raw::c_char;
    assert_eq!(log_entry(c_logger, message), 0);
    drop_logger(c_logger);
}
```
```c
#include "logger.h"
...
int main() {
    SimpleLogger *logger = NULL;
    if (create_simple_logger("test.log", &logger) == 0) {
        log_entry(logger, "Hello from C");
        drop_logger(logger); /*Needed to close handle, etc.*/
    } 
    ...
}
```

## 确保 unsafe 代码的正确性 {#ensuring-correctness-of-unsafe-code}
- 简而言之，使用 ```unsafe``` 需要深思熟虑
    - 始终记录代码的安全假设并与专家审查
    - 使用 cbindgen、Miri、Valgrind 等工具帮助验证正确性
    - **绝不要让 panic 跨越 FFI 边界展开**——这是 UB。在 FFI 入口点使用 `std::panic::catch_unwind`，或在 profile 中配置 `panic = "abort"`
    - 若结构体跨 FFI 共享，标记 `#[repr(C)]` 以保证 C 兼容内存布局
    - 查阅 https://doc.rust-lang.org/nomicon/intro.html（「Rustonomicon」—— unsafe Rust 的黑暗艺术）
    - 寻求内部专家帮助

### 验证工具：Miri 与 Valgrind

C++ 开发者熟悉 Valgrind 和 sanitizer。Rust 拥有这些**以及** Miri，对 Rust 特有的 UB 更精确：

| | **Miri** | **Valgrind** | **C++ sanitizer（ASan/MSan/UBSan）** |
|---|---------|-------------|--------------------------------------|
| **能捕获什么** | Rust 特有 UB：stacked borrows、无效 `enum` 判别式、未初始化读取、别名违规 | 内存泄漏、释放后使用、无效读/写、未初始化内存 | 缓冲区溢出、释放后使用、数据竞争、UB |
| **工作原理** | 解释 MIR（Rust 中级 IR）——无原生执行 | 在运行时检测编译后的二进制 | 编译期插桩 |
| **FFI 支持** | ❌ 无法跨越 FFI 边界（跳过 C 调用） | ✅ 适用于任何编译后的二进制，包括 FFI | ✅ 若 C 代码也用 sanitizer 编译则可用 |
| **速度** | 比原生慢约 100 倍 | 慢约 10–50 倍 | 慢约 2–5 倍 |
| **何时使用** | 纯 Rust `unsafe` 代码、数据结构不变式 | FFI 代码、完整二进制集成测试 | FFI 的 C/C++ 侧、性能敏感测试 |
| **捕获别名 bug** | ✅ Stacked Borrows 模型 | ❌ | 部分（TSan 用于数据竞争） |

**建议**：**两者都用**——纯 Rust unsafe 用 Miri，FFI 集成用 Valgrind：

- **Miri**——捕获 Valgrind 看不到的 Rust 特有 UB（别名违规、无效枚举值、stacked borrows）：
    ```
    rustup +nightly component add miri
    cargo +nightly miri test                    # Run all tests under Miri
    cargo +nightly miri test -- test_name       # Run a specific test
    ```
    > ⚠️ Miri 需要 nightly，无法执行 FFI 调用。将 unsafe Rust 逻辑隔离为可测试单元。

- **Valgrind**——你已熟悉的工具，适用于包括 FFI 在内的编译后二进制：
    ```
    sudo apt install valgrind
    cargo install cargo-valgrind
    cargo valgrind test                         # Run all tests under Valgrind
    ```
    > 捕获 FFI 代码中常见的 `Box::leak` / `Box::from_raw` 模式的泄漏。

- **cargo-careful**——在启用额外运行时检查的情况下运行测试（介于常规测试与 Miri 之间）：
    ```
    cargo install cargo-careful
    cargo +nightly careful test
    ```

## Unsafe Rust 总结
- ```cbindgen``` 是 (C) FFI 到 Rust 的优秀工具
    - 反向 FFI 接口使用 ```bindgen```（查阅详尽文档）
- **不要假设你的 unsafe 代码是正确的，或从 Safe Rust 使用它没问题。很容易犯错，即使看似正确的代码也可能因微妙原因而出错**
    - 使用工具验证正确性
    - 若仍有疑问，寻求专家意见
- 确保 ```unsafe``` 代码有明确记录假设及为何正确的注释
    - ```unsafe``` 代码的调用者也应有相应的 safety 注释，并遵守限制

# 练习：编写 Safe Rust FFI 包装 {#exercise-writing-a-safe-ffi-wrapper}

🔴 **挑战**——需要理解 unsafe 块、裸指针和 Safe Rust API 设计

- 围绕 `unsafe` FFI 风格函数编写 Safe Rust 包装。练习模拟调用将格式化字符串写入调用者提供缓冲区的 C 函数。
- **步骤 1**：实现 unsafe 函数 `unsafe_greet`，将问候语写入裸 `*mut u8` 缓冲区
- **步骤 2**：编写 safe 包装 `safe_greet`，分配 `Vec<u8>`，调用 unsafe 函数，返回 `String`
- **步骤 3**：为每个 unsafe 块添加适当的 `// Safety:` 注释

**Starter code:**
```rust
use std::fmt::Write as _;

/// Simulates a C function: writes "Hello, <name>!" into buffer.
/// Returns the number of bytes written (excluding null terminator).
/// # Safety
/// - `buf` must point to at least `buf_len` writable bytes
/// - `name` must be a valid pointer to a null-terminated C string
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // TODO: Build greeting, copy bytes into buf, return length
    // Hint: use std::ffi::CStr::from_ptr or iterate bytes manually
    todo!()
}

/// Safe wrapper — no unsafe in the public API
fn safe_greet(name: &str) -> Result<String, String> {
    // TODO: Allocate a Vec<u8> buffer, create a null-terminated name,
    // call unsafe_greet inside an unsafe block with Safety comment,
    // convert the result back to a String
    todo!()
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("Error: {e}"),
    }
    // Expected output: Hello, Rustacean!
}
```

<details><summary>Solution (click to expand)</summary>

```rust
use std::ffi::CStr;

/// Simulates a C function: writes "Hello, <name>!" into buffer.
/// Returns the number of bytes written, or -1 if buffer too small.
/// # Safety
/// - `buf` must point to at least `buf_len` writable bytes
/// - `name` must be a valid pointer to a null-terminated C string
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // Safety: caller guarantees name is a valid null-terminated string
    let name_cstr = unsafe { CStr::from_ptr(name as *const std::os::raw::c_char) };
    let name_str = match name_cstr.to_str() {
        Ok(s) => s,
        Err(_) => return -1,
    };
    let greeting = format!("Hello, {}!", name_str);
    if greeting.len() > buf_len {
        return -1;
    }
    // Safety: buf points to at least buf_len writable bytes (caller guarantee)
    unsafe {
        std::ptr::copy_nonoverlapping(greeting.as_ptr(), buf, greeting.len());
    }
    greeting.len() as isize
}

/// Safe wrapper — no unsafe in the public API
fn safe_greet(name: &str) -> Result<String, String> {
    let mut buffer = vec![0u8; 256];
    // Create a null-terminated version of name for the C API
    let name_with_null: Vec<u8> = name.bytes().chain(std::iter::once(0)).collect();

    // Safety: buffer has 256 writable bytes, name_with_null is null-terminated
    let bytes_written = unsafe {
        unsafe_greet(buffer.as_mut_ptr(), buffer.len(), name_with_null.as_ptr())
    };

    if bytes_written < 0 {
        return Err("Buffer too small or invalid name".to_string());
    }

    String::from_utf8(buffer[..bytes_written as usize].to_vec())
        .map_err(|e| format!("Invalid UTF-8: {e}"))
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("Error: {e}"),
    }
}
// Output:
// Hello, Rustacean!
```

</details>

----

