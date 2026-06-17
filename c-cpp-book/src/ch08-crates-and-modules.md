# Rust crate 与 module {#rust-crates-and-modules}

> **你将学到：** Rust 如何用 module 与 crate 组织代码——默认私有的可见性、`pub` 修饰符、workspace，以及 `crates.io` 生态。替代 C/C++ 头文件、`#include` 与 CMake 依赖管理。

- module 是 crate 内代码的基本组织单元
    - 每个源文件（.rs）是一个 module，可用 ```mod``` 创建嵌套 module
    - （子）module 中所有类型**默认私有**，除非显式标记为 ```pub```（公开），否则同一 crate 内也不对外可见。```pub``` 范围可进一步限制为 ```pub(crate)``` 等
    - 即使类型公开，也须用 ```use``` 导入才会在另一 module 作用域可见。子 module 可用 ```use super::``` 引用父作用域类型
    - 源文件（.rs）**不会**自动纳入 crate，除非在 ```main.rs```（可执行）或 ```lib.rs``` 中显式列出

# 练习：module 与函数 {#exercise-modules-and-functions}
- 修改 [hello world](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=522d86dbb8c4af71ff2ec081fb76aee7) 以调用另一函数
    - 如前所述，函数用 ```fn``` 定义。```->``` 声明函数返回值（默认为 void），类型为 ```u32```（无符号 32 位整数）
    - 函数按 module 作用域，即两个 module 中同名函数不会冲突
        - module 作用域适用于所有类型（例如 ```mod a { struct foo; }``` 中的 ```struct foo``` 与 ```mod b { struct foo; }``` 中的 ```b::foo``` 是不同类型）

**Starter code** — 补全函数：
```rust
mod math {
    // TODO: implement pub fn add(a: u32, b: u32) -> u32
}

fn greet(name: &str) -> String {
    // TODO: return "Hello, <name>! The secret number is <math::add(21,21)>"
    todo!()
}

fn main() {
    println!("{}", greet("Rustacean"));
}
```

<details><summary>Solution (click to expand)</summary>

```rust
mod math {
    pub fn add(a: u32, b: u32) -> u32 {
        a + b
    }
}

fn greet(name: &str) -> String {
    format!("Hello, {}! The secret number is {}", name, math::add(21, 21))
}

fn main() {
    println!("{}", greet("Rustacean"));
}
// Output: Hello, Rustacean! The secret number is 42
```

</details>


## Workspace 与 crate（包） {#workspaces-and-crates-packages}

- 任何有意义的 Rust 项目应用 workspace 组织组件 crate
    - workspace 是用于构建目标二进制的本地 crate 集合。workspace 根目录的 `Cargo.toml` 应指向各成员包（crate）

```toml
[workspace]
resolver = "2"
members = ["package1", "package2"]
```

```text
workspace_root/
|-- Cargo.toml      # Workspace configuration
|-- package1/
|   |-- Cargo.toml  # Package 1 configuration
|   `-- src/
|       `-- lib.rs  # Package 1 source code
|-- package2/
|   |-- Cargo.toml  # Package 2 configuration
|   `-- src/
|       `-- main.rs # Package 2 source code
```

---
## 练习：使用 workspace 与包依赖 {#exercise-using-workspaces-and-package-dependencies}
- 创建简单包并在 ```hello world``` 程序中使用
- 创建 workspace 目录
```bash
mkdir workspace
cd workspace
```
- 创建 Cargo.toml 并添加以下内容，创建空 workspace
```toml
[workspace]
resolver = "2"
members = []
```
- 添加包（```cargo new --lib``` 指定库而非可执行文件）
```bash
cargo new hello
cargo new --lib hellolib
```

## 练习：使用 workspace 与包依赖
- 查看 ```hello``` 与 ```hellolib``` 生成的 Cargo.toml。注意两者都已加入上层 ```Cargo.toml```
- ```hellolib``` 中存在 ```lib.rs``` 表示库包（定制选项见 https://doc.rust-lang.org/cargo/reference/cargo-targets.html）
- 在 ```hello``` 的 ```Cargo.toml``` 中添加对 ```hellolib``` 的依赖
```toml
[dependencies]
hellolib = {path = "../hellolib"}
```
- 使用 ```hellolib``` 的 ```add()```
```rust
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
```

<details><summary>Solution (click to expand)</summary>

The complete workspace setup:

```bash
# Terminal commands
mkdir workspace && cd workspace

# Create workspace Cargo.toml
cat > Cargo.toml << 'EOF'
[workspace]
resolver = "2"
members = ["hello", "hellolib"]
EOF

cargo new hello
cargo new --lib hellolib
```

```toml
# hello/Cargo.toml — add dependency
[dependencies]
hellolib = {path = "../hellolib"}
```

```rust
// hellolib/src/lib.rs — already has add() from cargo new --lib
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
```

```rust,ignore
// hello/src/main.rs
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
// Output: Hello, world! 42
```

</details>

# 使用 crates.io 社区 crate {#using-community-crates-from-cratesio}
- Rust 有活跃的社区 crate 生态（见 https://crates.io/）
    - Rust 哲学是保持标准库精简，将功能外包给社区 crate
    - 使用社区 crate 无硬性规则，但经验法则是确保 crate 成熟度合理（版本号可反映）且仍在维护。若有疑问可咨询内部资源
- ```crates.io``` 上每个 crate 有主版本与次版本
    - crate 应遵循此处定义的 ```SemVer``` 主/次版本指南：https://doc.rust-lang.org/cargo/reference/semver.html
    - 简言之：同一 minor 版本内不应有破坏性变更。例如 v0.11 须与 v0.15 兼容（v0.20 可有破坏性变更）

# Crate 依赖与 SemVer {#crates-dependencies-and-semver}
- crate 可依赖特定版本、特定 minor/major 版本，或不限制。下列示例为在 ```Cargo.toml``` 中声明对 ```rand``` crate 的依赖
- 至少 ```0.10.0```，但 ```< 0.11.0``` 的任意版本均可
```toml
[dependencies]
rand = { version = "0.10.0"}
```
- 仅 ```0.10.0```，不接受其他版本
```toml
[dependencies]
rand = { version = "=0.10.0"}
```
- 不限制；```cargo``` 将选择最新版本
```toml
[dependencies]
rand = { version = "*"}
```
- 参考：https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
----
# 练习：使用 rand crate {#exercise-using-the-rand-crate}
- 修改 ```helloworld``` 示例以打印随机数
- 用 ```cargo add rand``` 添加依赖
- 以 ```https://docs.rs/rand/latest/rand/``` 为 API 参考

**Starter code** — 运行 `cargo add rand` 后在 `main.rs` 中添加：
```rust,ignore
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    // TODO: Generate and print a random u32 in 1..=100
    // TODO: Generate and print a random bool
    // TODO: Generate and print a random f64
}
```

<details><summary>Solution (click to expand)</summary>

```rust
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    let n: u32 = rng.random_range(1..=100);
    println!("Random number (1-100): {n}");

    // Generate a random boolean
    let b: bool = rng.random();
    println!("Random bool: {b}");

    // Generate a random float between 0.0 and 1.0
    let f: f64 = rng.random();
    println!("Random float: {f:.4}");
}
```

</details>

# Cargo.toml 与 Cargo.lock {#cargotoml-and-cargolock}
- 如前所述，Cargo.lock 由 Cargo.toml 自动生成
    - Cargo.lock 的主要目的是保证可重现构建。例如若 ```Cargo.toml``` 指定 ```0.10.0```，cargo 可选 ```< 0.11.0``` 的任意版本
    - Cargo.lock 包含构建时使用的 rand crate **具体**版本
    - 建议将 ```Cargo.lock``` 纳入 git 仓库以保证可重现构建

## Cargo test 功能 {#cargo-test-feature}
- Rust 单元测试通常与源码同文件（按惯例），常分组为独立 module
    - 测试代码从不纳入实际二进制。这由 ```cfg```（配置）特性实现。配置可用于平台特定代码（如 ```Linux``` vs ```Windows```）
    - 测试用 ```cargo test``` 执行。参考：https://doc.rust-lang.org/reference/conditional-compilation.html

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
// Will be included only during testing
#[cfg(test)]
mod tests {
    use super::*; // This makes all types in the parent scope visible
    #[test]
    fn it_works() {
        let result = add(2, 2); // Alternatively, super::add(2, 2);
        assert_eq!(result, 4);
    }
}
```

# 其他 Cargo 功能 {#other-cargo-features}
- ```cargo``` 还有其他实用功能，包括：
    - ```cargo clippy``` 是优秀的 Rust 代码 lint 工具。一般应修复警告（或确有理由时极少抑制）
    - ```cargo format``` 运行 ```rustfmt``` 格式化源码。保证入库代码格式统一，终结风格争论
    - ```cargo doc``` 可从 ```///``` 风格注释生成文档。```crates.io``` 上所有 crate 文档均用此方法生成

### 构建配置：控制优化

C 中向 `gcc`/`clang` 传 `-O0`、`-O2`、`-Os`、`-flto`。Rust 在 `Cargo.toml` 中配置构建配置：

```toml
# Cargo.toml — build profile configuration

[profile.dev]
opt-level = 0          # No optimization (fast compile, like -O0)
debug = true           # Full debug symbols (like -g)

[profile.release]
opt-level = 3          # Maximum optimization (like -O3)
lto = "fat"            # Link-Time Optimization (like -flto)
strip = true           # Strip symbols (like the strip command)
codegen-units = 1      # Single codegen unit — slower compile, better optimization
panic = "abort"        # No unwind tables (smaller binary)
```

| C/GCC 标志 | Cargo.toml 键 | 取值 |
|------------|---------------|--------|
| `-O0` / `-O2` / `-O3` | `opt-level` | `0`、`1`、`2`、`3`、`"s"`、`"z"` |
| `-flto` | `lto` | `false`、`"thin"`、`"fat"` |
| `-g` / 无 `-g` | `debug` | `true`、`false`、`"line-tables-only"` |
| `strip` 命令 | `strip` | `"none"`、`"debuginfo"`、`"symbols"`、`true`/`false` |
| — | `codegen-units` | `1` = 最优优化、编译最慢 |

```bash
cargo build              # Uses [profile.dev]
cargo build --release    # Uses [profile.release]
```

### 构建脚本（`build.rs`）：链接 C 库

C 中用 Makefile 或 CMake 链接库并运行代码生成。
Rust 在 crate 根目录使用 `build.rs`：

```rust
// build.rs — runs before compiling the crate

fn main() {
    // Link a system C library (like -lbmc_ipmi in gcc)
    println!("cargo::rustc-link-lib=bmc_ipmi");

    // Where to find the library (like -L/usr/lib/bmc)
    println!("cargo::rustc-link-search=/usr/lib/bmc");

    // Re-run if the C header changes
    println!("cargo::rerun-if-changed=wrapper.h");
}
```

甚至可从 Rust crate 直接编译 C 源文件：

```toml
# Cargo.toml
[build-dependencies]
cc = "1"  # C compiler integration
```

```rust
// build.rs
fn main() {
    cc::Build::new()
        .file("src/c_helpers/ipmi_raw.c")
        .include("/usr/include/bmc")
        .compile("ipmi_raw");   // Produces libipmi_raw.a, linked automatically
    println!("cargo::rerun-if-changed=src/c_helpers/ipmi_raw.c");
}
```

| C / Make / CMake | Rust `build.rs` |
|-----------------|-----------------|
| `-lfoo` | `println!("cargo::rustc-link-lib=foo")` |
| `-L/path` | `println!("cargo::rustc-link-search=/path")` |
| 编译 C 源 | `cc::Build::new().file("foo.c").compile("foo")` |
| 生成代码 | 写入 `$OUT_DIR`，再 `include!()` |

### 交叉编译

C 中交叉编译需安装独立工具链（`arm-linux-gnueabihf-gcc`）并配置 Make/CMake。Rust 中：

```bash
# Install a cross-compilation target
rustup target add aarch64-unknown-linux-gnu

# Cross-compile
cargo build --target aarch64-unknown-linux-gnu --release
```

在 `.cargo/config.toml` 中指定链接器：

```toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

| C 交叉编译 | Rust 等价 |
|-----------------|-----------------|
| `apt install gcc-aarch64-linux-gnu` | `rustup target add aarch64-unknown-linux-gnu` + 安装链接器 |
| `CC=aarch64-linux-gnu-gcc make` | `.cargo/config.toml` `[target.X] linker = "..."` |
| `#ifdef __aarch64__` | `#[cfg(target_arch = "aarch64")]` |
| 独立 Makefile 目标 | `cargo build --target ...` |

### 特性标志：条件编译

C 用 `#ifdef` 与 `-DFOO` 做条件编译。Rust 在 `Cargo.toml` 中定义特性标志：

```toml
# Cargo.toml
[features]
default = ["json"]         # Enabled by default
json = ["dep:serde_json"]  # Optional dependency
verbose = []               # Flag with no dependency
gpu = ["dep:cuda-sys"]     # Optional GPU support
```

```rust
// Code gated on features:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose"))]
macro_rules! verbose {
    ($($arg:tt)*) => {}; // Compiles to nothing
}
```

| C 预处理器 | Rust 特性标志 |
|---------------|-------------------|
| `gcc -DDEBUG` | `cargo build --features verbose` |
| `#ifdef DEBUG` | `#[cfg(feature = "verbose")]` |
| `#define MAX 100` | `const MAX: u32 = 100;` |
| `#ifdef __linux__` | `#[cfg(target_os = "linux")]` |

### 集成测试 vs 单元测试

单元测试与代码同处，使用 `#[cfg(test)]`。**集成测试**位于 `tests/`，仅测试 crate 的**公开 API**：

```rust
// tests/smoke_test.rs — no #[cfg(test)] needed
use my_crate::parse_config;

#[test]
fn parse_valid_config() {
    let config = parse_config("test_data/valid.json").unwrap();
    assert_eq!(config.max_retries, 5);
}
```

| 方面 | 单元测试（`#[cfg(test)]`） | 集成测试（`tests/`） |
|--------|----------------------------|------------------------------|
| 位置 | 与代码同文件 | 独立 `tests/` 目录 |
| 访问 | 私有 + 公开项 | **仅公开 API** |
| 运行命令 | `cargo test` | `cargo test --test smoke_test` |


### 测试模式与策略

C 固件团队常用 CUnit、CMocka 或自定义框架，样板很多。Rust 内置测试 harness 能力更强。本节涵盖生产代码所需模式。

#### `#[should_panic]` — 测试预期失败

```rust
// Test that certain conditions cause panics (like C's assert failures)
#[test]
#[should_panic(expected = "index out of bounds")]
fn test_bounds_check() {
    let v = vec![1, 2, 3];
    let _ = v[10];  // Should panic
}

#[test]
#[should_panic(expected = "temperature exceeds safe limit")]
fn test_thermal_shutdown() {
    fn check_temperature(celsius: f64) {
        if celsius > 105.0 {
            panic!("temperature exceeds safe limit: {celsius}°C");
        }
    }
    check_temperature(110.0);
}
```

#### `#[ignore]` — 慢速或依赖硬件的测试

```rust
// Mark tests that require special conditions (like C's #ifdef HARDWARE_TEST)
#[test]
#[ignore = "requires GPU hardware"]
fn test_gpu_ecc_scrub() {
    // This test only runs on machines with GPUs
    // Run with: cargo test -- --ignored
    // Run with: cargo test -- --include-ignored  (runs ALL tests)
}
```

#### 返回 `Result` 的测试（替代 `unwrap` 链）

```rust
// Instead of many unwrap() calls that hide the actual failure:
#[test]
fn test_config_parsing() -> Result<(), Box<dyn std::error::Error>> {
    let json = r#"{"hostname": "node-01", "port": 8080}"#;
    let config: ServerConfig = serde_json::from_str(json)?;  // ? instead of unwrap()
    assert_eq!(config.hostname, "node-01");
    assert_eq!(config.port, 8080);
    Ok(())  // Test passes if we reach here without error
}
```

#### 用构建函数做测试夹具

C 用 `setUp()`/`tearDown()`。Rust 用辅助函数与 `Drop`：

```rust
struct TestFixture {
    temp_dir: std::path::PathBuf,
    config: Config,
}

impl TestFixture {
    fn new() -> Self {
        let temp_dir = std::env::temp_dir().join(format!("test_{}", std::process::id()));
        std::fs::create_dir_all(&temp_dir).unwrap();
        let config = Config {
            log_dir: temp_dir.clone(),
            max_retries: 3,
            ..Default::default()
        };
        Self { temp_dir, config }
    }
}

impl Drop for TestFixture {
    fn drop(&mut self) {
        // Automatic cleanup — like C's tearDown() but can't be forgotten
        let _ = std::fs::remove_dir_all(&self.temp_dir);
    }
}

#[test]
fn test_with_fixture() {
    let fixture = TestFixture::new();
    // Use fixture.config, fixture.temp_dir...
    assert!(fixture.temp_dir.exists());
    // fixture is automatically dropped here → cleanup runs
}
```

#### 用 Trait 模拟硬件接口

C 中模拟硬件需预处理器技巧或函数指针替换。
Rust 中 Trait 很自然：

```rust
// Production trait for IPMI communication
trait IpmiTransport {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String>;
}

// Real implementation (used in production)
struct RealIpmi { /* BMC connection details */ }
impl IpmiTransport for RealIpmi {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String> {
        // Actually talks to BMC hardware
        todo!("Real IPMI call")
    }
}

// Mock implementation (used in tests)
struct MockIpmi {
    responses: std::collections::HashMap<u8, Vec<u8>>,
}
impl IpmiTransport for MockIpmi {
    fn send_command(&self, cmd: u8, _data: &[u8]) -> Result<Vec<u8>, String> {
        self.responses.get(&cmd)
            .cloned()
            .ok_or_else(|| format!("No mock response for cmd 0x{cmd:02x}"))
    }
}

// Generic function that works with both real and mock
fn read_sensor_temperature(transport: &dyn IpmiTransport) -> Result<f64, String> {
    let response = transport.send_command(0x2D, &[])?;
    if response.len() < 2 {
        return Err("Response too short".into());
    }
    Ok(response[0] as f64 + (response[1] as f64 / 256.0))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_temperature_reading() {
        let mut mock = MockIpmi { responses: std::collections::HashMap::new() };
        mock.responses.insert(0x2D, vec![72, 128]); // 72.5°C

        let temp = read_sensor_temperature(&mock).unwrap();
        assert!((temp - 72.5).abs() < 0.01);
    }

    #[test]
    fn test_short_response() {
        let mock = MockIpmi { responses: std::collections::HashMap::new() };
        // No response configured → error
        assert!(read_sensor_temperature(&mock).is_err());
    }
}
```

#### 用 `proptest` 做基于属性的测试

不测试具体值，而测试**对所有输入都成立的不变式**：

```rust
// Cargo.toml: [dev-dependencies] proptest = "1"
use proptest::prelude::*;

fn parse_sensor_id(s: &str) -> Option<u32> {
    s.strip_prefix("sensor_")?.parse().ok()
}

fn format_sensor_id(id: u32) -> String {
    format!("sensor_{id}")
}

proptest! {
    #[test]
    fn roundtrip_sensor_id(id in 0u32..10000) {
        // Property: format then parse should give back the original
        let formatted = format_sensor_id(id);
        let parsed = parse_sensor_id(&formatted);
        prop_assert_eq!(parsed, Some(id));
    }

    #[test]
    fn parse_rejects_garbage(s in "[^s].*") {
        // Property: strings not starting with 's' should never parse
        let result = parse_sensor_id(&s);
        prop_assert!(result.is_none());
    }
}
```

#### C vs Rust 测试对照

| C 测试 | Rust 等价 |
|-----------|----------------|
| `CUnit`、`CMocka`、自定义框架 | 内置 `#[test]` + `cargo test` |
| `setUp()` / `tearDown()` | 构建函数 + `Drop` Trait |
| `#ifdef TEST` 模拟函数 | 基于 Trait 的依赖注入 |
| `assert(x == y)` | `assert_eq!(x, y)`，自动 diff 输出 |
| 独立测试可执行文件 | 同一二进制，`#[cfg(test)]` 条件编译 |
| `valgrind --leak-check=full ./test` | `cargo test`（默认内存安全）+ `cargo miri test` |
| 代码覆盖率：`gcov` / `lcov` | `cargo tarpaulin` 或 `cargo llvm-cov` |
| 测试发现：手动注册 | 自动——任意 `#[test]` 函数都会被发现 |


