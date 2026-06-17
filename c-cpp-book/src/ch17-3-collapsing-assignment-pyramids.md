## 用闭包折叠赋值金字塔 {#collapsing-assignment-pyramids-with-closures}

> **你将学到：** Rust 基于表达式的语法与闭包如何将 C++ 中深层嵌套的 `if/else` 校验链压平为清晰、线性的代码。

- C++ 在赋值变量时常需要多分支 `if/else` 链，尤其在涉及校验或回退逻辑时。Rust 基于表达式的语法与闭包将这些压平为扁平、线性的代码。

### 模式 1：用 `if` 表达式进行元组赋值
```cpp
// C++ — three variables set across a multi-block if/else chain
uint32_t fault_code;
const char* der_marker;
const char* action;
if (is_c44ad) {
    fault_code = 32709; der_marker = "CSI_WARN"; action = "No action";
} else if (error.is_hardware_error()) {
    fault_code = 67956; der_marker = "CSI_ERR"; action = "Replace GPU";
} else {
    fault_code = 32709; der_marker = "CSI_WARN"; action = "No action";
}
```

```rust
// Rust equivalent:accel_fieldiag.rs
// Single expression assigns all three at once:
let (fault_code, der_marker, recommended_action) = if is_c44ad {
    (32709u32, "CSI_WARN", "No action")
} else if error.is_hardware_error() {
    (67956u32, "CSI_ERR", "Replace GPU")
} else {
    (32709u32, "CSI_WARN", "No action")
};
```

### 模式 2：IIFE（立即调用函数表达式）处理可失败链
```cpp
// C++ — pyramid of doom for JSON navigation
std::string get_part_number(const nlohmann::json& root) {
    if (root.contains("SystemInfo")) {
        auto& sys = root["SystemInfo"];
        if (sys.contains("BaseboardFru")) {
            auto& bb = sys["BaseboardFru"];
            if (bb.contains("ProductPartNumber")) {
                return bb["ProductPartNumber"].get<std::string>();
            }
        }
    }
    return "UNKNOWN";
}
```

```rust
// Rust equivalent:framework.rs
// Closure + ? operator collapses the pyramid into linear code:
let part_number = (|| -> Option<String> {
    let path = self.args.sysinfo.as_ref()?;
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    let ppn = json
        .get("SystemInfo")?
        .get("BaseboardFru")?
        .get("ProductPartNumber")?
        .as_str()?;
    Some(ppn.to_string())
})()
.unwrap_or_else(|| "UNKNOWN".to_string());
```
闭包创建一个 `Option<String>` 作用域，任一步失败时 `?` 会提前退出。`.unwrap_or_else()` 在末尾一次性提供回退值。

### 模式 3：迭代器链替代手动循环 + push_back
```cpp
// C++ — manual loop with intermediate variables
std::vector<std::tuple<std::vector<std::string>, std::string, std::string>> gpu_info;
for (const auto& [key, info] : gpu_pcie_map) {
    std::vector<std::string> bdfs;
    // ... parse bdf_path into bdfs
    std::string serial = info.serial_number.value_or("UNKNOWN");
    std::string model = info.model_number.value_or(model_name);
    gpu_info.push_back({bdfs, serial, model});
}
```

```rust
// Rust equivalent:peripherals.rs
// Single chain: values() → map → collect
let gpu_info: Vec<(Vec<String>, String, String, String)> = self
    .gpu_pcie_map
    .values()
    .map(|info| {
        let bdfs: Vec<String> = info.bdf_path
            .split(')')
            .filter(|s| !s.is_empty())
            .map(|s| s.trim_start_matches('(').to_string())
            .collect();
        let serial = info.serial_number.clone()
            .unwrap_or_else(|| "UNKNOWN".to_string());
        let model = info.model_number.clone()
            .unwrap_or_else(|| model_name.to_string());
        let gpu_bdf = format!("{}:{}:{}.{}",
            info.bdf.segment, info.bdf.bus, info.bdf.device, info.bdf.function);
        (bdfs, serial, model, gpu_bdf)
    })
    .collect();
```

### 模式 4：`.filter().collect()` 替代循环 + `if (condition) continue`
```cpp
// C++
std::vector<TestResult*> failures;
for (auto& t : test_results) {
    if (!t.is_pass()) {
        failures.push_back(&t);
    }
}
```

```rust
// Rust — from accel_diag/src/healthcheck.rs
pub fn failed_tests(&self) -> Vec<&TestResult> {
    self.test_results.iter().filter(|t| !t.is_pass()).collect()
}
```

### 小结：何时使用各模式
| **C++ 模式** | **Rust 替代** | **主要收益** |
|----------------|---------------------|-----------------|
| 多分支变量赋值 | `let (a, b) = if ... { } else { };` | 所有变量原子绑定 |
| 嵌套 `if (contains)` 金字塔 | 带 `?` 的 IIFE 闭包 | 线性、扁平、早退 |
| `for` 循环 + `push_back` | `.iter().map(\|\|).collect()` | 无需中间可变 `Vec` |
| `for` + `if (cond) continue` | `.iter().filter(\|\|).collect()` | 声明式意图 |
| `for` + `if + break`（找第一个） | `.iter().find_map(\|\|)` | 一次遍历完成搜索与变换 |

----

# 综合练习：诊断事件流水线 {#capstone-exercise-diagnostic-event-pipeline}

🔴 **挑战** — 综合练习，涵盖枚举、Trait、迭代器、错误处理与泛型

本综合练习整合枚举、Trait（特征）、迭代器、错误处理与泛型。你将构建一个简化的诊断事件处理流水线，类似生产 Rust 代码中的模式。

**要求：**
1. 定义带 `Display` 的 `enum Severity { Info, Warning, Critical }`，以及包含 `source: String`、`severity: Severity`、`message: String`、`fault_code: u32` 的 `struct DiagEvent`
2. 定义带方法 `fn should_include(&self, event: &DiagEvent) -> bool` 的 `trait EventFilter`
3. 实现两个过滤器：`SeverityFilter`（仅包含 >= 给定严重级别的事件）和 `SourceFilter`（仅包含来自特定源字符串的事件）
4. 编写函数 `fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String>`，返回通过**全部**过滤器的事件的格式化报告行
5. 编写 `fn parse_event(line: &str) -> Result<DiagEvent, String>`，解析 `"source:severity:fault_code:message"` 形式的行（错误输入返回 `Err`）

**起始代码：**
```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Severity {
    Info,
    Warning,
    Critical,
}

impl fmt::Display for Severity {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        todo!()
    }
}

#[derive(Debug, Clone)]
struct DiagEvent {
    source: String,
    severity: Severity,
    message: String,
    fault_code: u32,
}

trait EventFilter {
    fn should_include(&self, event: &DiagEvent) -> bool;
}

struct SeverityFilter {
    min_severity: Severity,
}
// TODO: impl EventFilter for SeverityFilter

struct SourceFilter {
    source: String,
}
// TODO: impl EventFilter for SourceFilter

fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String> {
    // TODO: Filter events that pass ALL filters, format as
    // "[SEVERITY] source (FC:fault_code): message"
    todo!()
}

fn parse_event(line: &str) -> Result<DiagEvent, String> {
    // Parse "source:severity:fault_code:message"
    // Return Err for invalid input
    todo!()
}

fn main() {
    let raw_lines = vec![
        "accel_diag:Critical:67956:ECC uncorrectable error detected",
        "nic_diag:Warning:32709:Link speed degraded",
        "accel_diag:Info:10001:Self-test passed",
        "cpu_diag:Critical:55012:Thermal throttling active",
        "accel_diag:Warning:32710:PCIe link width reduced",
    ];

    // Parse all lines, collect successes and report errors
    let events: Vec<DiagEvent> = raw_lines.iter()
        .filter_map(|line| match parse_event(line) {
            Ok(e) => Some(e),
            Err(e) => { eprintln!("Parse error: {e}"); None }
        })
        .collect();

    // Apply filters: only Critical+Warning events from accel_diag
    let sev_filter = SeverityFilter { min_severity: Severity::Warning };
    let src_filter = SourceFilter { source: "accel_diag".to_string() };
    let filters: Vec<&dyn EventFilter> = vec![&sev_filter, &src_filter];

    let report = process_events(&events, &filters);
    for line in &report {
        println!("{line}");
    }
    println!("--- {} event(s) matched ---", report.len());
}
```

<details><summary>解答（点击展开）</summary>

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Severity {
    Info,
    Warning,
    Critical,
}

impl fmt::Display for Severity {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Severity::Info => write!(f, "INFO"),
            Severity::Warning => write!(f, "WARNING"),
            Severity::Critical => write!(f, "CRITICAL"),
        }
    }
}

impl Severity {
    fn from_str(s: &str) -> Result<Self, String> {
        match s {
            "Info" => Ok(Severity::Info),
            "Warning" => Ok(Severity::Warning),
            "Critical" => Ok(Severity::Critical),
            other => Err(format!("Unknown severity: {other}")),
        }
    }
}

#[derive(Debug, Clone)]
struct DiagEvent {
    source: String,
    severity: Severity,
    message: String,
    fault_code: u32,
}

trait EventFilter {
    fn should_include(&self, event: &DiagEvent) -> bool;
}

struct SeverityFilter {
    min_severity: Severity,
}

impl EventFilter for SeverityFilter {
    fn should_include(&self, event: &DiagEvent) -> bool {
        event.severity >= self.min_severity
    }
}

struct SourceFilter {
    source: String,
}

impl EventFilter for SourceFilter {
    fn should_include(&self, event: &DiagEvent) -> bool {
        event.source == self.source
    }
}

fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String> {
    events.iter()
        .filter(|e| filters.iter().all(|f| f.should_include(e)))
        .map(|e| format!("[{}] {} (FC:{}): {}", e.severity, e.source, e.fault_code, e.message))
        .collect()
}

fn parse_event(line: &str) -> Result<DiagEvent, String> {
    let parts: Vec<&str> = line.splitn(4, ':').collect();
    if parts.len() != 4 {
        return Err(format!("Expected 4 colon-separated fields, got {}", parts.len()));
    }
    let fault_code = parts[2].parse::<u32>()
        .map_err(|e| format!("Invalid fault code '{}': {e}", parts[2]))?;
    Ok(DiagEvent {
        source: parts[0].to_string(),
        severity: Severity::from_str(parts[1])?,
        fault_code,
        message: parts[3].to_string(),
    })
}

fn main() {
    let raw_lines = vec![
        "accel_diag:Critical:67956:ECC uncorrectable error detected",
        "nic_diag:Warning:32709:Link speed degraded",
        "accel_diag:Info:10001:Self-test passed",
        "cpu_diag:Critical:55012:Thermal throttling active",
        "accel_diag:Warning:32710:PCIe link width reduced",
    ];

    let events: Vec<DiagEvent> = raw_lines.iter()
        .filter_map(|line| match parse_event(line) {
            Ok(e) => Some(e),
            Err(e) => { eprintln!("Parse error: {e}"); None }
        })
        .collect();

    let sev_filter = SeverityFilter { min_severity: Severity::Warning };
    let src_filter = SourceFilter { source: "accel_diag".to_string() };
    let filters: Vec<&dyn EventFilter> = vec![&sev_filter, &src_filter];

    let report = process_events(&events, &filters);
    for line in &report {
        println!("{line}");
    }
    println!("--- {} event(s) matched ---", report.len());
}
// Output:
// [CRITICAL] accel_diag (FC:67956): ECC uncorrectable error detected
// [WARNING] accel_diag (FC:32710): PCIe link width reduced
// --- 2 event(s) matched ---
```

</details>

----

