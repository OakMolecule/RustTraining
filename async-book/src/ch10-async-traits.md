# 10. 异步 Trait 🟡

> **你将学到：**
> - 为何 Trait 中的异步方法稳定化花了多年
> - RPITIT：原生异步 Trait 方法（Rust 1.75+）
> - `dyn` 分发挑战与通过 `trait_variant` 的 `Send` 约束
> - 异步闭包（Rust 1.85+）：`async Fn()` 与 `async FnOnce()`

```mermaid
graph TD
    subgraph "异步 Trait 方案"
        direction TB
        RPITIT["RPITIT（Rust 1.75+）<br/>trait 中 async fn<br/>仅静态分发"]
        VARIANT["trait_variant<br/>自动生成 Send 变体<br/>仅静态分发"]
        BOXED["Box&lt;dyn Future&gt;<br/>手动装箱<br/>到处可用"]
        CLOSURE["异步闭包（1.85+）<br/>async Fn() / async FnOnce()<br/>回调与中间件"]
    end

    RPITIT -->|"需要 Send？"| VARIANT
    RPITIT -->|"需要 dyn？"| BOXED
    CLOSURE -->|"替代"| BOXED

    style RPITIT fill:#d4efdf,stroke:#27ae60,color:#000
    style VARIANT fill:#e8f4f8,stroke:#2980b9,color:#000
    style BOXED fill:#fef9e7,stroke:#f39c12,color:#000
    style CLOSURE fill:#e8daef,stroke:#8e44ad,color:#000
```

## 历史：为何花了这么久

Trait 中的异步方法是 Rust 多年来最被请求的功能。问题在于：

```rust
// This didn't compile until Rust 1.75 (Dec 2023):
trait DataStore {
    async fn get(&self, key: &str) -> Option<String>;
}
// Why? Because async fn returns `impl Future<Output = T>`,
// and `impl Trait` in trait return position wasn't supported.
```

根本挑战：当 Trait 方法返回 `impl Future` 时，每个实现返回*不同的具体类型*。编译器需要知道返回类型的大小，但 Trait 方法是动态分发的。

### RPITIT：Trait 返回位置的 Impl Trait

自 Rust 1.75 起，静态分发下可以直接这样写：

```rust
trait DataStore {
    async fn get(&self, key: &str) -> Option<String>;
    // Desugars to:
    // fn get(&self, key: &str) -> impl Future<Output = Option<String>>;
}

struct InMemoryStore {
    data: std::collections::HashMap<String, String>,
}

impl DataStore for InMemoryStore {
    async fn get(&self, key: &str) -> Option<String> {
        self.data.get(key).cloned()
    }
}

// ✅ Works with generics (static dispatch):
async fn lookup<S: DataStore>(store: &S, key: &str) {
    if let Some(val) = store.get(key).await {
        println!("{key} = {val}");
    }
}
```

### dyn 分发与 Send 约束

限制：不能直接使用 `dyn DataStore`，因为编译器不知道返回的 future 的大小：

```rust
// ❌ Doesn't work:
// async fn lookup_dyn(store: &dyn DataStore, key: &str) { ... }
// Error: the trait `DataStore` is not dyn-compatible because method `get`
//        is `async`

// ✅ Workaround: Return a boxed future
trait DynDataStore {
    fn get(&self, key: &str) -> Pin<Box<dyn Future<Output = Option<String>> + Send + '_>>;
}
```

**Send 问题**：在多线程运行时中，spawn 的任务必须是 `Send`。但异步 Trait 方法不会自动加上 `Send` 约束：

```rust
trait Worker {
    async fn run(self); // Future might or might not be Send
}

struct MyWorker;

impl Worker for MyWorker {
    async fn run(self) {
        // If this uses !Send types, the future is !Send
        let rc = std::rc::Rc::new(42);
        some_work().await;
        println!("{rc}");
    }
}

// ❌ This fails because the future is !Send (Rc is !Send):
// tokio::spawn(worker.run()); // Requires Send + 'static
//
// Note: We use `self` (owned) here because tokio::spawn also
// requires 'static — a future borrowing &self can't be 'static.
// Even without Rc, `async fn run(&self)` wouldn't be spawnable.
```

### trait_variant crate

`trait_variant` crate（来自 Rust 异步工作组）会自动生成 `Send` 变体：

```rust
// Cargo.toml: trait-variant = "0.1"

#[trait_variant::make(SendDataStore: Send)]
trait DataStore {
    async fn get(&self, key: &str) -> Option<String>;
    async fn set(&self, key: &str, value: String);
}

// Now you have two traits:
// - DataStore: no Send bound on the futures
// - SendDataStore: all futures are Send
// Both have the same methods, implementors implement DataStore
// and get SendDataStore for free if their futures are Send.

// Use SendDataStore when you need to spawn tasks:
async fn spawn_lookup<S: SendDataStore + 'static>(store: Arc<S>) {
    tokio::spawn(async move {
        store.get("key").await;
    });
}

// ⚠️ Note: trait_variant does NOT enable dyn dispatch.
// The generated trait still uses `impl Future`, so `dyn SendDataStore`
// is not object-safe. For dyn dispatch, you still need manual boxing
// (see the Box::pin approach above) or the `async-trait` crate.
```

### 速查：异步 Trait

| 方案 | 静态分发 | 动态分发 | Send | 语法开销 |
|----------|:---:|:---:|:---:|---|
| Trait 内原生 `async fn` | ✅ | ❌ | 隐式 | 无 |
| `trait_variant` | ✅ | ❌ | 显式 | `#[trait_variant::make]` |
| 手动 `Box::pin` | ✅ | ✅ | 显式 | 高 |
| `async-trait` crate | ✅ | ✅ | `#[async_trait]` | 中（过程宏） |

> **建议**：新代码（Rust 1.75+）使用原生异步 Trait。需要为 spawn 任务加 `Send` 约束时用
> `trait_variant`。需要 `dyn`
> 分发时用手动 `Box::pin` 或 `async-trait` crate。原生
> 方案在静态分发下零成本。

### 异步闭包（Rust 1.85+）

自 Rust 1.85 起，**异步闭包**（async closures）已稳定——捕获环境并返回 future 的闭包：

```rust
// Before 1.85: awkward workaround
let urls = vec!["https://a.com", "https://b.com"];
let fetchers: Vec<_> = urls.iter().map(|url| {
    let url = url.to_string();
    // Returns a non-async closure that returns an async block
    move || async move { reqwest::get(&url).await }
}).collect();

// After 1.85: async closures just work
let fetchers: Vec<_> = urls.iter().map(|url| {
    async move || { reqwest::get(url).await }
    // ↑ This is an async closure — captures url, returns a Future
}).collect();
```

异步闭包实现新的 `AsyncFn`、`AsyncFnMut`、`AsyncFnOnce` Trait，分别对应 `Fn`、`FnMut`、`FnOnce`：

```rust
// Generic function accepting an async closure
async fn retry<F>(max: usize, f: F) -> Result<String, Error>
where
    F: AsyncFn() -> Result<String, Error>,
{
    for _ in 0..max {
        if let Ok(val) = f().await {
            return Ok(val);
        }
    }
    f().await
}
```

> **迁移提示**：若代码使用 `Fn() -> impl Future<Output = T>`，
> 可考虑改为 `AsyncFn() -> T` 以获得更清晰的签名。

<details>
<summary><strong>🏋️ 练习：设计异步服务 Trait</strong>（点击展开）</summary>

**挑战**：设计带异步 `get` 和 `set` 方法的 `Cache` Trait。实现两次：一次用 `HashMap`（内存），一次用模拟 Redis 后端（用 `tokio::time::sleep` 模拟网络延迟）。写一个对两者都适用的泛型函数。

<details>
<summary>🔑 解答</summary>

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::time::{sleep, Duration};

trait Cache {
    async fn get(&self, key: &str) -> Option<String>;
    async fn set(&self, key: &str, value: String);
}

// --- In-memory implementation ---
struct MemoryCache {
    store: Mutex<HashMap<String, String>>,
}

impl MemoryCache {
    fn new() -> Self {
        MemoryCache {
            store: Mutex::new(HashMap::new()),
        }
    }
}

impl Cache for MemoryCache {
    async fn get(&self, key: &str) -> Option<String> {
        self.store.lock().await.get(key).cloned()
    }

    async fn set(&self, key: &str, value: String) {
        self.store.lock().await.insert(key.to_string(), value);
    }
}

// --- Simulated Redis implementation ---
struct RedisCache {
    store: Mutex<HashMap<String, String>>,
    latency: Duration,
}

impl RedisCache {
    fn new(latency_ms: u64) -> Self {
        RedisCache {
            store: Mutex::new(HashMap::new()),
            latency: Duration::from_millis(latency_ms),
        }
    }
}

impl Cache for RedisCache {
    async fn get(&self, key: &str) -> Option<String> {
        sleep(self.latency).await; // Simulate network round-trip
        self.store.lock().await.get(key).cloned()
    }

    async fn set(&self, key: &str, value: String) {
        sleep(self.latency).await;
        self.store.lock().await.insert(key.to_string(), value);
    }
}

// --- Generic function working with any Cache ---
async fn cache_demo<C: Cache>(cache: &C, label: &str) {
    cache.set("greeting", "Hello, async!".into()).await;
    let val = cache.get("greeting").await;
    println!("[{label}] greeting = {val:?}");
}

#[tokio::main]
async fn main() {
    let mem = MemoryCache::new();
    cache_demo(&mem, "memory").await;

    let redis = RedisCache::new(50);
    cache_demo(&redis, "redis").await;
}
```

**要点**：同一泛型函数通过静态分发适用于两种实现。无需装箱，无分配开销。若要在多线程运行时 spawn 这些 future，添加 `trait_variant::make(SendCache: Send)` 以获得 `Send` 约束。动态分发请用手动 `Box::pin` 或 `async-trait` crate。

</details>
</details>

> **要点回顾 — 异步 Trait**
> - 自 Rust 1.75 起，可在 Trait 中直接写 `async fn`（无需 `#[async_trait]` crate）
> - `trait_variant::make` 自动为 spawn 任务生成 `Send` 变体（仅静态分发）
> - 异步闭包（`async Fn()`）在 1.85 稳定——用于回调与中间件
> - 性能敏感代码优先静态分发（`<S: Service>`）而非 `dyn`

> **另见：** [第 13 章 — 生产模式](ch13-production-patterns.md) 了解 Tower 的 `Service` Trait，[第 6 章 — 手动构建 Future](ch06-building-futures-by-hand.md) 了解手动 Trait 实现

***

