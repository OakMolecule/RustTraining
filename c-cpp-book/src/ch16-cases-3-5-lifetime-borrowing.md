# 案例研究 3：框架通信 → 生命周期借用 {#case-study-3-framework-communication--lifetime-borrowing}

> **你将学到：** 如何将 C++ 裸指针框架通信模式转换为 Rust 基于生命周期的借用系统，在保持零成本抽象的同时消除悬垂指针风险。

## C++ 模式：指向框架的裸指针 {#the-c-pattern-raw-pointer-to-framework}
```cpp
// C++ original: Every diagnostic module stores a raw pointer to the framework
class DiagBase {
protected:
    DiagFramework* m_pFramework;  // Raw pointer — who owns this?
public:
    DiagBase(DiagFramework* fw) : m_pFramework(fw) {}
    
    void LogEvent(uint32_t code, const std::string& msg) {
        m_pFramework->GetEventLog()->Record(code, msg);  // Hope it's still alive!
    }
};
// Problem: m_pFramework is a raw pointer with no lifetime guarantee
// If framework is destroyed while modules still reference it → UB
```

## Rust 方案：带生命周期借用的 `DiagContext` {#the-rust-solution-diagcontext-with-lifetime-borrowing}
```rust
// Example: module.rs — Borrow, don't store

/// Context passed to diagnostic modules during execution.
/// The lifetime 'a guarantees the framework outlives the context.
pub struct DiagContext<'a> {
    pub der_log: &'a mut EventLogManager,
    pub config: &'a ModuleConfig,
    pub framework_opts: &'a HashMap<String, String>,
}

/// Modules receive context as a parameter — never store framework pointers
pub trait DiagModule {
    fn id(&self) -> &str;
    fn execute(&mut self, ctx: &mut DiagContext) -> DiagResult<()>;
    fn pre_execute(&mut self, _ctx: &mut DiagContext) -> DiagResult<()> {
        Ok(())
    }
    fn post_execute(&mut self, _ctx: &mut DiagContext) -> DiagResult<()> {
        Ok(())
    }
}
```

### 关键洞察 {#key-insight}
- C++ 模块**存储**指向框架的指针（危险：框架先被销毁怎么办？）
- Rust 模块**接收**上下文作为函数参数 — 借用检查器保证调用期间框架仍然存活
- 无裸指针、无生命周期歧义、无需「希望它还活着」

----

# 案例研究 4：God object → 可组合状态 {#case-study-4-god-object--composable-state}

## C++ 模式：单体框架类 {#the-c-pattern-monolithic-framework-class}
```cpp
// C++ original: The framework is god object
class DiagFramework {
    // Health-monitor trap processing
    std::vector<AlertTriggerInfo> m_alertTriggers;
    std::vector<WarnTriggerInfo> m_warnTriggers;
    bool m_healthMonHasBootTimeError;
    uint32_t m_healthMonActionCounter;
    
    // GPU diagnostics
    std::map<uint32_t, GpuPcieInfo> m_gpuPcieMap;
    bool m_isRecoveryContext;
    bool m_healthcheckDetectedDevices;
    // ... 30+ more GPU-related fields
    
    // PCIe tree
    std::shared_ptr<CPcieTreeLinux> m_pPcieTree;
    
    // Event logging
    CEventLogMgr* m_pEventLogMgr;
    
    // ... several other methods
    void HandleGpuEvents();
    void HandleNicEvents();
    void RunGpuDiag();
    // Everything depends on everything
};
```

## Rust 方案：可组合状态结构体 {#the-rust-solution-composable-state-structs}
```rust
// Example: main.rs — State decomposed into focused structs

#[derive(Default)]
struct HealthMonitorState {
    alert_triggers: Vec<AlertTriggerInfo>,
    warn_triggers: Vec<WarnTriggerInfo>,
    health_monitor_action_counter: u32,
    health_monitor_has_boot_time_error: bool,
    // Only health-monitor-related fields
}

#[derive(Default)]
struct GpuDiagState {
    gpu_pcie_map: HashMap<u32, GpuPcieInfo>,
    is_recovery_context: bool,
    healthcheck_detected_devices: bool,
    // Only GPU-related fields
}

/// The framework composes these states rather than owning everything flat
struct DiagFramework {
    ctx: DiagContext,             // Execution context
    args: Args,                   // CLI arguments
    pcie_tree: Option<DeviceTree>,  // No shared_ptr needed
    event_log_mgr: EventLogManager,   // Owned, not raw pointer
    fc_manager: FcManager,        // Fault code management
    health: HealthMonitorState,   // Health-monitor state — its own struct
    gpu: GpuDiagState,           // GPU state — its own struct
}
```

### 关键洞察 {#key-insight-2}
- **可测试性**：每个状态结构体可独立单元测试
- **可读性**：`self.health.alert_triggers` vs `m_alertTriggers` — 所有权清晰
- **放心重构**：修改 `GpuDiagState` 不会意外影响健康监控处理
- **无方法堆砌**：只需健康监控状态的函数接受 `&mut HealthMonitorState`，而非整个框架

----

# 案例研究 5：Trait 对象 — 何时才合适 {#case-study-5-trait-objects--when-they-are-right}

- 并非一切都应是枚举！**诊断模块插件系统**是 Trait 对象的正当用例
- 原因：诊断模块**对扩展开放** — 可添加新模块而无需修改框架

```rust
// Example: framework.rs — Vec<Box<dyn DiagModule>> is correct here
pub struct DiagFramework {
    modules: Vec<Box<dyn DiagModule>>,        // Runtime polymorphism
    pre_diag_modules: Vec<Box<dyn DiagModule>>,
    event_log_mgr: EventLogManager,
    // ...
}

impl DiagFramework {
    /// Register a diagnostic module — any type implementing DiagModule
    pub fn register_module(&mut self, module: Box<dyn DiagModule>) {
        info!("Registering module: {}", module.id());
        self.modules.push(module);
    }
}
```

### 何时使用哪种模式 {#when-to-use-each-pattern}

| **用例** | **模式** | **原因** |
|-------------|-----------|--------|
| 编译期已知的固定变体集合 | `enum` + `match` | 穷尽检查、无 vtable |
| 硬件事件类型（Degrade、Fatal、Boot 等） | `enum GpuEventKind` | 所有变体已知，性能重要 |
| PCIe 设备类型（GPU、NIC、Switch 等） | `enum PcieDeviceKind` | 固定集合，各变体数据不同 |
| 插件/模块系统（对扩展开放） | `Box<dyn Trait>` | 无需修改框架即可添加新模块 |
| 测试 mock | `Box<dyn Trait>` | 注入测试替身 |

### 练习：翻译前先思考 {#exercise-think-before-you-translate}
给定以下 C++ 代码：
```cpp
class Shape { public: virtual double area() = 0; };
class Circle : public Shape { double r; double area() override { return 3.14*r*r; } };
class Rect : public Shape { double w, h; double area() override { return w*h; } };
std::vector<std::unique_ptr<Shape>> shapes;
```
**问题**：Rust 翻译应使用 `enum Shape` 还是 `Vec<Box<dyn Shape>>`？

<details><summary>解答（点击展开）</summary>

**答案**：`enum Shape` — 因为形状集合是**封闭的**（编译期已知）。仅当用户可在运行时添加新形状类型时，才使用 `Box<dyn Shape>`。

```rust
// Correct Rust translation:
enum Shape {
    Circle { r: f64 },
    Rect { w: f64, h: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { r } => std::f64::consts::PI * r * r,
            Shape::Rect { w, h } => w * h,
        }
    }
}

fn main() {
    let shapes: Vec<Shape> = vec![
        Shape::Circle { r: 5.0 },
        Shape::Rect { w: 3.0, h: 4.0 },
    ];
    for shape in &shapes {
        println!("Area: {:.2}", shape.area());
    }
}
// Output:
// Area: 78.54
// Area: 12.00
```

</details>

----

# 翻译指标与经验总结 {#translation-metrics-and-lessons-learned}

## 我们学到了什么 {#what-we-learned}
1. **默认使用枚举分发** — 在约 10 万行 C++ 中，仅约 25 处 `Box<dyn Trait>` 真正必要（插件系统、测试 mock）。其余约 900 个虚方法变为带 `match` 的枚举
2. **Arena 模式消除引用循环** — `shared_ptr` 与 `enable_shared_from_this` 是所有权不清的症状。先想清楚谁**拥有**数据
3. **传递上下文，不要存储指针** — 生命周期有界的 `DiagContext<'a>` 比在每个模块里存 `Framework*` 更安全、更清晰
4. **分解 god object** — 若结构体有 30+ 字段，多半是 3–4 个结构体穿同一件大衣
5. **编译器是你的结对伙伴** — 约 400 次 `dynamic_cast` 意味着约 400 次潜在运行时失败。Rust 中零 `dynamic_cast` 等价物意味着零运行时类型错误

## 最难的部分 {#the-hardest-parts}
- **生命周期注解**：习惯了裸指针后，把借用写对需要时间 — 但一旦编译通过，就是对的
- **与借用检查器较劲**：想在两处同时持有 `&mut self`。解法：将状态分解为独立结构体
- **抵制字面翻译**：处处写 `Vec<Box<dyn Base>>` 的诱惑。问自己：「这组变体是封闭的吗？」→ 若是，用枚举

## 给 C++ 团队的建议 {#recommendation-for-c-teams}
1. 从小而独立的模块开始（不要先动 god object）
2. 先翻译数据结构，再翻译行为
3. 让编译器引导你 — 错误信息非常出色
4. 在 `dyn Trait` 之前先考虑 `enum`
5. 用 [Rust playground](https://play.rust-lang.org/) 在集成前原型验证模式

----

