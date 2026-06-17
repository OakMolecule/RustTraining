## 面向 C++ 程序员的测试模式

> **你将学到：** Rust 内置测试框架——`#[test]`、`#[should_panic]`、返回 `Result` 的测试、测试数据的构建器模式、基于 Trait 的 mock、`proptest` 属性测试、`insta` 快照测试，以及集成测试组织。零配置测试，替代 Google Test + CMake。

C++ 测试通常依赖外部框架（Google Test、Catch2、Boost.Test）与复杂构建集成。Rust 测试框架**内置于语言与工具链**——无依赖、无 CMake 集成、无测试运行器配置。

### `#[test]` 之外的测试属性

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn basic_pass() {
        assert_eq!(2 + 2, 4);
    }

    // Expect a panic — equivalent to GTest's EXPECT_DEATH
    #[test]
    #[should_panic]
    fn out_of_bounds_panics() {
        let v = vec![1, 2, 3];
        let _ = v[10]; // Panics — test passes
    }

    // Expect a panic with a specific message substring
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn specific_panic_message() {
        let v = vec![1, 2, 3];
        let _ = v[10];
    }

    // Tests that return Result<(), E> — use ? instead of unwrap()
    #[test]
    fn test_with_result() -> Result<(), String> {
        let value: u32 = "42".parse().map_err(|e| format!("{e}"))?;
        assert_eq!(value, 42);
        Ok(())
    }

    // Ignore slow tests by default — run with `cargo test -- --ignored`
    #[test]
    #[ignore]
    fn slow_integration_test() {
        std::thread::sleep(std::time::Duration::from_secs(10));
    }
}
```

```bash
cargo test                          # Run all non-ignored tests
cargo test -- --ignored             # Run only ignored tests
cargo test -- --include-ignored     # Run ALL tests including ignored
cargo test test_name                # Run tests matching a name pattern
cargo test -- --nocapture           # Show println! output during tests
cargo test -- --test-threads=1      # Run tests serially (for shared state)
```

### 测试辅助：测试数据的构建器模式

C++ 中用 Google Test fixture（`class MyTest : public ::testing::Test`）。
Rust 中用构建函数或 `Default` Trait：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Builder function — creates test data with sensible defaults
    fn make_gpu_event(severity: Severity, fault_code: u32) -> DiagEvent {
        DiagEvent {
            source: "accel_diag".to_string(),
            severity,
            message: format!("Test event FC:{fault_code}"),
            fault_code,
        }
    }

    // Reusable test fixture — a set of pre-built events
    fn sample_events() -> Vec<DiagEvent> {
        vec![
            make_gpu_event(Severity::Critical, 67956),
            make_gpu_event(Severity::Warning, 32709),
            make_gpu_event(Severity::Info, 10001),
        ]
    }

    #[test]
    fn filter_critical_events() {
        let events = sample_events();
        let critical: Vec<_> = events.iter()
            .filter(|e| e.severity == Severity::Critical)
            .collect();
        assert_eq!(critical.len(), 1);
        assert_eq!(critical[0].fault_code, 67956);
    }
}
```

### 用 Trait 做 mock

C++ 中 mock 需 Google Mock 或手动虚函数覆盖。
Rust 中为依赖定义 Trait，在测试中替换实现：

```rust
// Production trait
trait SensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String>;
}

// Production implementation
struct HwSensorReader;
impl SensorReader for HwSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        // Real hardware call...
        Ok(72.5)
    }
}

// Test mock — returns predictable values
#[cfg(test)]
struct MockSensorReader {
    temperatures: std::collections::HashMap<u32, f64>,
}

#[cfg(test)]
impl SensorReader for MockSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        self.temperatures.get(&sensor_id)
            .copied()
            .ok_or_else(|| format!("Unknown sensor {sensor_id}"))
    }
}

// Function under test — generic over the reader
fn check_overtemp(reader: &impl SensorReader, ids: &[u32], threshold: f64) -> Vec<u32> {
    ids.iter()
        .filter(|&&id| reader.read_temperature(id).unwrap_or(0.0) > threshold)
        .copied()
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn detect_overtemp_sensors() {
        let mut mock = MockSensorReader { temperatures: Default::default() };
        mock.temperatures.insert(0, 72.5);
        mock.temperatures.insert(1, 91.0);  // Over threshold
        mock.temperatures.insert(2, 65.0);

        let hot = check_overtemp(&mock, &[0, 1, 2], 80.0);
        assert_eq!(hot, vec![1]);
    }
}
```

### 测试中的临时文件与目录

C++ 测试常用平台相关临时目录。Rust 有 `tempfile`：

```rust
// Cargo.toml: [dev-dependencies]
// tempfile = "3"

#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::NamedTempFile;
    use std::io::Write;

    #[test]
    fn parse_config_from_file() -> Result<(), Box<dyn std::error::Error>> {
        // Create a temp file that's auto-deleted when dropped
        let mut file = NamedTempFile::new()?;
        writeln!(file, r#"{{"sku": "ServerNode", "level": "Quick"}}"#)?;

        let config = load_config(file.path().to_str().unwrap())?;
        assert_eq!(config.sku, "ServerNode");
        Ok(())
        // file is deleted here — no cleanup code needed
    }
}
```

### 用 `proptest` 做基于属性的测试

不写具体用例，而描述**对所有输入都应成立的不变式**。
`proptest` 生成随机输入并找最小失败用例：

```rust
// Cargo.toml: [dev-dependencies]
// proptest = "1"

#[cfg(test)]
mod tests {
    use proptest::prelude::*;

    fn parse_and_format(n: u32) -> String {
        format!("{n}")
    }

    proptest! {
        #[test]
        fn roundtrip_u32(n: u32) {
            let formatted = parse_and_format(n);
            let parsed: u32 = formatted.parse().unwrap();
            prop_assert_eq!(n, parsed);
        }

        #[test]
        fn string_contains_no_null(s in "[a-zA-Z0-9 ]{0,100}") {
            prop_assert!(!s.contains('\0'));
        }
    }
}
```

### 用 `insta` 做快照测试

对产生复杂输出（JSON、格式化字符串）的测试，`insta` 自动生成并管理参考快照：

```rust
// Cargo.toml: [dev-dependencies]
// insta = { version = "1", features = ["json"] }

#[cfg(test)]
mod tests {
    use insta::assert_json_snapshot;

    #[test]
    fn der_entry_format() {
        let entry = DerEntry {
            fault_code: 67956,
            component: "GPU".to_string(),
            message: "ECC error detected".to_string(),
        };
        // First run: creates a snapshot file in tests/snapshots/
        // Subsequent runs: compares against the saved snapshot
        assert_json_snapshot!(entry);
    }
}
```

```bash
cargo insta test              # Run tests and review new/changed snapshots
cargo insta review            # Interactive review of snapshot changes
```

### C++ vs Rust 测试对照

| **C++（Google Test）** | **Rust** | **说明** |
|----------------------|---------|----------|
| `TEST(Suite, Name) { }` | `#[test] fn name() { }` | 无需 suite/类层次 |
| `ASSERT_EQ(a, b)` | `assert_eq!(a, b)` | 内置宏，无需框架 |
| `ASSERT_NEAR(a, b, eps)` | `assert!((a - b).abs() < eps)` | 或使用 `approx` crate |
| `EXPECT_THROW(expr, type)` | `#[should_panic(expected = "...")]` | 或 `catch_unwind` 精细控制 |
| `EXPECT_DEATH(expr, "msg")` | `#[should_panic(expected = "msg")]` | |
| `class Fixture : public ::testing::Test` | 构建函数 + `Default` | 无需继承 |
| Google Mock `MOCK_METHOD` | Trait + 测试 impl | 更显式，无宏魔法 |
| `INSTANTIATE_TEST_SUITE_P`（参数化） | `proptest!` 或宏生成测试 | |
| `SetUp()` / `TearDown()` | 通过 `Drop` 的 RAII——清理自动 | 测试结束变量 drop |
| 独立测试二进制 + CMake | `cargo test`——零配置 | |
| `ctest --output-on-failure` | `cargo test -- --nocapture` | |

----

### 集成测试：`tests/` 目录

单元测试在 `#[cfg(test)]` module 中与代码同处。**集成测试**位于 crate 根目录独立 `tests/`，像外部使用者一样测试库公开 API：

```
my_crate/
├── src/
│   └── lib.rs          # Your library code
├── tests/
│   ├── smoke.rs        # Each .rs file is a separate test binary
│   ├── regression.rs
│   └── common/
│       └── mod.rs      # Shared test helpers (NOT a test itself)
└── Cargo.toml
```

```rust
// tests/smoke.rs — tests your crate as an external user would
use my_crate::DiagEngine;  // Only public API is accessible

#[test]
fn engine_starts_successfully() {
    let engine = DiagEngine::new("test_config.json");
    assert!(engine.is_ok());
}

#[test]
fn engine_rejects_invalid_config() {
    let engine = DiagEngine::new("nonexistent.json");
    assert!(engine.is_err());
}
```

```rust
// tests/common/mod.rs — shared helpers, NOT compiled as a test binary
pub fn setup_test_environment() -> tempfile::TempDir {
    let dir = tempfile::tempdir().unwrap();
    std::fs::write(dir.path().join("config.json"), r#"{"log_level": "debug"}"#).unwrap();
    dir
}
```

```rust
// tests/regression.rs — can use shared helpers
mod common;

#[test]
fn regression_issue_42() {
    let env = common::setup_test_environment();
    let engine = my_crate::DiagEngine::new(
        env.path().join("config.json").to_str().unwrap()
    );
    assert!(engine.is_ok());
}
```

**运行集成测试：**
```bash
cargo test                          # Runs unit AND integration tests
cargo test --test smoke             # Run only tests/smoke.rs
cargo test --test regression        # Run only tests/regression.rs
cargo test --lib                    # Run ONLY unit tests (skip integration)
```

> **与单元测试的关键区别**：集成测试不能访问私有函数或 `pub(crate)` 项。这迫使你验证公开 API 是否足够——是有价值的设计信号。C++ 术语下，类似只测公开头文件、无 `friend` 访问。

----

