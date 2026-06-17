## 日志与追踪：syslog/printf → `log` + `tracing` {#logging-and-tracing-ecosystem}

> **你将学到：** Rust 的双层日志架构（门面 + 后端）、`log` 与 `tracing` crate、带 span 的结构化日志，以及如何替代 `printf`/`syslog` 调试。

C++ 诊断代码通常使用 `printf`、`syslog` 或自定义日志框架。
Rust 有标准化的双层日志架构：**门面** crate（`log` 或
`tracing`）与**后端**（实际日志实现）。

### `log` 门面 — Rust 通用日志 API

`log` crate 提供与 syslog 严重级别对应的宏。库使用
`log` 宏；可执行文件选择后端：

```rust
// Cargo.toml
// [dependencies]
// log = "0.4"
// env_logger = "0.11"    # One of many backends

use log::{info, warn, error, debug, trace};

fn check_sensor(id: u32, temp: f64) {
    trace!("Reading sensor {id}");           // Finest granularity
    debug!("Sensor {id} raw value: {temp}"); // Development-time detail

    if temp > 85.0 {
        warn!("Sensor {id} high temperature: {temp}°C");
    }
    if temp > 95.0 {
        error!("Sensor {id} CRITICAL: {temp}°C — initiating shutdown");
    }
    info!("Sensor {id} check complete");     // Normal operation
}

fn main() {
    // Initialize the backend — typically done once in main()
    env_logger::init();  // Controlled by RUST_LOG env var

    check_sensor(0, 72.5);
    check_sensor(1, 91.0);
}
```

```bash
# Control log level via environment variable
RUST_LOG=debug cargo run          # Show debug and above
RUST_LOG=warn cargo run           # Show only warn and error
RUST_LOG=my_crate=trace cargo run # Per-module filtering
RUST_LOG=my_crate::gpu=debug,warn cargo run  # Mix levels
```

### C++ 对比

| C++ | Rust（`log`） | 说明 |
|-----|-------------|-------|
| `printf("DEBUG: %s\n", msg)` | `debug!("{msg}")` | 编译期检查格式 |
| `syslog(LOG_ERR, "...")` | `error!("...")` | 由后端决定输出位置 |
| 日志调用外包 `#ifdef DEBUG` | 在 max_level 下 `trace!` / `debug!` 被编译掉 | 禁用时零成本 |
| 自定义 `Logger::log(level, msg)` | `log::info!("...")` — 所有 crate 同一 API | 通用门面，可换后端 |
| 按文件控制日志详细度 | `RUST_LOG=crate::module=level` | 基于环境变量，无需重编译 |

### `tracing` crate — 带 span 的结构化日志

`tracing` 在 `log` 基础上扩展了**结构化字段**与 **span**（带时间的作用域）。
对需要跟踪上下文的诊断代码尤其有用：

```rust
// Cargo.toml
// [dependencies]
// tracing = "0.1"
// tracing-subscriber = { version = "0.3", features = ["env-filter"] }

use tracing::{info, warn, error, instrument, info_span};

#[instrument(skip(data), fields(gpu_id = gpu_id, data_len = data.len()))]
fn run_gpu_test(gpu_id: u32, data: &[u8]) -> Result<(), String> {
    info!("Starting GPU test");

    let span = info_span!("ecc_check", gpu_id);
    let _guard = span.enter();  // All logs inside this scope include gpu_id

    if data.is_empty() {
        error!(gpu_id, "No test data provided");
        return Err("empty data".to_string());
    }

    // Structured fields — machine-parseable, not just string interpolation
    info!(
        gpu_id,
        temp_celsius = 72.5,
        ecc_errors = 0,
        "ECC check passed"
    );

    Ok(())
}

fn main() {
    // Initialize tracing subscriber
    tracing_subscriber::fmt()
        .with_env_filter("debug")  // Or use RUST_LOG env var
        .with_target(true)          // Show module path
        .with_thread_ids(true)      // Show thread IDs
        .init();

    let _ = run_gpu_test(0, &[1, 2, 3]);
}
```

使用 `tracing-subscriber` 时的输出：
```rust
2026-02-15T10:30:00.123Z DEBUG ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}: my_crate: Starting GPU test
2026-02-15T10:30:00.124Z  INFO ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}:ecc_check{gpu_id=0}: my_crate: ECC check passed gpu_id=0 temp_celsius=72.5 ecc_errors=0
```

### `#[instrument]` — 自动创建 span

`#[instrument]` 属性自动以函数名及其参数创建 span：

```rust
use tracing::instrument;

#[instrument]
fn parse_sel_record(record_id: u16, sensor_type: u8, data: &[u8]) -> Result<(), String> {
    // Every log inside this function automatically includes:
    // record_id, sensor_type, and data (if Debug)
    tracing::debug!("Parsing SEL record");
    Ok(())
}

// skip: exclude large/sensitive args from the span
// fields: add computed fields
#[instrument(skip(raw_buffer), fields(buf_len = raw_buffer.len()))]
fn decode_ipmi_response(raw_buffer: &[u8]) -> Result<Vec<u8>, String> {
    tracing::trace!("Decoding {} bytes", raw_buffer.len());
    Ok(raw_buffer.to_vec())
}
```

### `log` 与 `tracing` — 选哪个

| 方面 | `log` | `tracing` |
|--------|-------|-----------|
| **复杂度** | 简单 — 5 个宏 | 更丰富 — span、字段、instrument |
| **结构化数据** | 仅字符串插值 | 键值字段：`info!(gpu_id = 0, "msg")` |
| **计时 / span** | 无 | 有 — `#[instrument]`、`span.enter()` |
| **异步支持** | 基础 | 一等公民 — span 跨 `.await` 传播 |
| **兼容性** | 通用门面 | 与 `log` 兼容（有 `log` 桥接） |
| **适用场景** | 简单应用、库 | 诊断工具、异步代码、可观测性 |

> **建议**：生产级诊断类项目（带结构化输出的诊断工具）用 `tracing`。
> 依赖要少的简单库用 `log`。
> `tracing` 含兼容层，使用 `log`
> 宏的库在 `tracing` subscriber 下仍可工作。

### 后端选项

| 后端 Crate | 输出 | 适用场景 |
|--------------|--------|----------|
| `env_logger` | stderr，彩色 | 开发、简单 CLI 工具 |
| `tracing-subscriber` | stderr，格式化 | 生产环境配合 `tracing` |
| `syslog` | 系统 syslog | Linux 系统服务 |
| `tracing-journald` | systemd journal | systemd 管理的服务 |
| `tracing-appender` | 轮转日志文件 | 长期运行的守护进程 |
| `tracing-opentelemetry` | OpenTelemetry 收集器 | 分布式追踪 |

----

