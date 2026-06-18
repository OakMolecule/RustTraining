# 12. 常见陷阱 🔴

> **你将学到：**
> - 9 种常见异步 Rust bug 及各自的修复方法
> - 为何阻塞执行器（executor）是头号错误（以及 `spawn_blocking` 如何修复）
> - 取消（cancellation）风险：Future 在 `.await` 中途被丢弃时会发生什么
> - 调试：`tokio-console`、`tracing`、`#[instrument]`
> - 测试：`#[tokio::test]`、`time::pause()`、基于 Trait 的 mock

## 阻塞执行器

异步 Rust 的头号错误：在异步执行器线程上运行阻塞代码。这会饿死其他任务。

```rust
// ❌ WRONG: Blocks the entire executor thread
async fn bad_handler() -> String {
    let data = std::fs::read_to_string("big_file.txt").unwrap(); // BLOCKS!
    process(&data)
}

// ✅ CORRECT: Offload blocking work to a dedicated thread pool
async fn good_handler() -> String {
    let data = tokio::task::spawn_blocking(|| {
        std::fs::read_to_string("big_file.txt").unwrap()
    }).await.unwrap();
    process(&data)
}

// ✅ ALSO CORRECT: Use tokio's async fs
async fn also_good_handler() -> String {
    let data = tokio::fs::read_to_string("big_file.txt").await.unwrap();
    process(&data)
}
```

```mermaid
graph TB
    subgraph "❌ 在执行器上阻塞调用"
        T1_BAD["线程 1: std::fs::read()<br/>🔴 阻塞 500ms"]
        T2_BAD["线程 2: 处理请求<br/>🟢 独自工作"]
        TASKS_BAD["100 个待处理任务<br/>⏳ 被饿死"]
        T1_BAD -->|"无法 poll"| TASKS_BAD
    end

    subgraph "✅ spawn_blocking"
        T1_GOOD["线程 1: poll Future<br/>🟢 可用"]
        T2_GOOD["线程 2: poll Future<br/>🟢 可用"]
        BT["阻塞池线程:<br/>std::fs::read()<br/>🔵 独立线程池"]
        TASKS_GOOD["100 个任务<br/>✅ 全部推进"]
        T1_GOOD -->|"poll"| TASKS_GOOD
        T2_GOOD -->|"poll"| TASKS_GOOD
    end
```

### std::thread::sleep 与 tokio::time::sleep

```rust
// ❌ WRONG: Blocks the executor thread for 5 seconds
async fn bad_delay() {
    std::thread::sleep(Duration::from_secs(5)); // Thread can't poll anything else!
}

// ✅ CORRECT: Yields to the executor, other tasks can run
async fn good_delay() {
    tokio::time::sleep(Duration::from_secs(5)).await; // Non-blocking!
}
```

### 在 `.await` 期间持有 MutexGuard

```rust
use std::sync::Mutex; // std Mutex — NOT async-aware

// ⚠️ RISKY: MutexGuard held across .await
async fn bad_mutex(data: &Mutex<Vec<String>>) {
    let mut guard = data.lock().unwrap();
    guard.push("item".into());
    some_io().await; // Guard is held here — blocks other threads from locking!
    guard.push("another".into());
}
// NOTE: This compiles! std::sync::MutexGuard is !Send, but the compiler only
// enforces Send on the Future when you pass it to something that requires it
// (e.g., tokio::spawn). Calling bad_mutex(...).await directly compiles fine.
// However, tokio::spawn(bad_mutex(data)) will fail with a Send bound error.
```

**为何这通常是问题**——但并非总是：

在 `.await` 期间持有 `std::sync::Mutex` 会在 I/O 持续时间内阻塞 **OS 线程**，使执行器无法在该线程上 poll 其他任务。对于短临界区这是浪费；对于长 I/O 则是性能陷阱。

**然而**，有些合法场景下你 *必须* 在 `.await` 期间持有锁——就像数据库事务在读取与提交之间持有锁一样。丢弃并重新获取锁会引入 **TOCTOU（time-of-check to time-of-use，检查时间与使用时间）竞态**：另一个任务可以在你的两个临界区之间修改数据。正确修复取决于具体用例：

```rust
// OPTION 1: Scope the guard — works when operations are independent
async fn scoped_mutex(data: &Mutex<Vec<String>>) {
    {
        let mut guard = data.lock().unwrap();
        guard.push("item".into());
    } // Guard dropped here
    some_io().await; // Lock is released — other tasks can proceed
    {
        let mut guard = data.lock().unwrap();
        guard.push("another".into());
    }
}
// ⚠️ Careful: another task can lock + modify the Vec between the two sections.
//    This is fine if the two pushes are independent, but wrong if "another"
//    depends on state set by "item".

// OPTION 2: Use tokio::sync::Mutex — holds lock across .await without
//           blocking the OS thread. Best when you need transactional
//           read-modify-write across an await point.
use tokio::sync::Mutex as AsyncMutex;

async fn async_mutex(data: &AsyncMutex<Vec<String>>) {
    let mut guard = data.lock().await; // Async lock — doesn't block the thread
    guard.push("item".into());
    some_io().await; // OK — tokio Mutex guard is Send
    guard.push("another".into());
    // Guard held the whole time — no TOCTOU race, no thread blocked.
}
```

> **何时使用哪种 Mutex**：
> - `std::sync::Mutex`：短临界区，内部无 `.await`
> - `tokio::sync::Mutex`：需要在 `.await` 点持有锁时（事务语义、避免 TOCTOU）
> - `parking_lot::Mutex`：`std` 的直接替代，更快更小，仍不可 `.await`
>
> **经验法则**：不要盲目在 `.await` 两侧拆分临界区。问自己两半是否真正独立。若不是——若后半依赖前半的状态——使用 `tokio::sync::Mutex` 或重新设计数据流。

### 取消风险

丢弃 Future 会取消它——但这可能使事物处于不一致状态：

```rust
// ❌ DANGEROUS: Resource leak on cancellation
async fn transfer(from: &Account, to: &Account, amount: u64) {
    from.debit(amount).await;  // If cancelled HERE...
    to.credit(amount).await;   // ...money vanishes!
}

// ✅ SAFE: Make operations atomic or use compensation
async fn safe_transfer(from: &Account, to: &Account, amount: u64) -> Result<(), Error> {
    // Use a database transaction (all-or-nothing)
    let tx = db.begin_transaction().await?;
    tx.debit(from, amount).await?;
    tx.credit(to, amount).await?;
    tx.commit().await?; // Only commits if everything succeeded
    Ok(())
}

// ✅ ALSO SAFE: Use tokio::select! with cancellation awareness
tokio::select! {
    result = transfer(from, to, amount) => {
        // Transfer completed
    }
    _ = shutdown_signal() => {
        // Don't cancel mid-transfer — let it finish
        // Or: roll back explicitly
    }
}
```

### 没有异步 Drop

Rust 的 `Drop` Trait 是同步的——你 **不能** 在 `drop()` 内 `.await`。这是常见的困惑来源：

```rust
struct DbConnection { /* ... */ }

impl Drop for DbConnection {
    fn drop(&mut self) {
        // ❌ Can't do this — drop() is sync!
        // self.connection.shutdown().await;

        // ✅ Workaround 1: Spawn a cleanup task (fire-and-forget)
        let conn = self.connection.take();
        tokio::spawn(async move {
            let _ = conn.shutdown().await;
        });

        // ✅ Workaround 2: Use a synchronous close
        // self.connection.blocking_close();
    }
}
```

**最佳实践**：提供显式的 `async fn close(self)` 方法，并文档说明调用者应使用它。仅将 `Drop` 作为安全网，而非主要清理路径。

### select! 公平性与饥饿

```rust
use tokio::sync::mpsc;

// ❌ UNFAIR: busy_stream always wins, slow_stream starves
async fn unfair(mut fast: mpsc::Receiver<i32>, mut slow: mpsc::Receiver<i32>) {
    loop {
        tokio::select! {
            Some(v) = fast.recv() => println!("fast: {v}"),
            Some(v) = slow.recv() => println!("slow: {v}"),
            // If both are ready, tokio randomly picks one.
            // But if `fast` is ALWAYS ready, `slow` rarely gets polled.
        }
    }
}

// ✅ FAIR: Use biased select or drain in batches
async fn fair(mut fast: mpsc::Receiver<i32>, mut slow: mpsc::Receiver<i32>) {
    loop {
        tokio::select! {
            biased; // Always check in order — explicit priority

            Some(v) = slow.recv() => println!("slow: {v}"),  // Priority!
            Some(v) = fast.recv() => println!("fast: {v}"),
        }
    }
}
```

### 意外的顺序执行

```rust
// ❌ SEQUENTIAL: Takes 2 seconds total
async fn slow() {
    let a = fetch("url_a").await; // 1 second
    let b = fetch("url_b").await; // 1 second (waits for a to finish first!)
}

// ✅ CONCURRENT: Takes 1 second total
async fn fast() {
    let (a, b) = tokio::join!(
        fetch("url_a"), // Both start immediately
        fetch("url_b"),
    );
}

// ✅ ALSO CONCURRENT: Using let + join
async fn also_fast() {
    let fut_a = fetch("url_a"); // Create future (lazy — not started yet)
    let fut_b = fetch("url_b"); // Create future
    let (a, b) = tokio::join!(fut_a, fut_b); // NOW both run concurrently
}
```

> **陷阱**：`let a = fetch(url).await; let b = fetch(url).await;` 是顺序执行！
> 第二个 `.await` 要等第一个完成后才开始。并发请用 `join!` 或 `spawn`。

## 案例研究：调试挂起的生产服务

真实场景：服务前 10 分钟正常处理请求，随后停止响应。日志无错误。CPU 为 0%。

**诊断步骤：**

1. **附加 `tokio-console`**——显示 200+ 个任务卡在 `Pending` 状态
2. **查看任务详情**——全部等待同一个 `Mutex::lock().await`
3. **根因**——一个任务在 `.await` 期间持有 `std::sync::MutexGuard` 并 panic，毒化了 mutex。其他任务在 `lock().unwrap()` 时全部失败

**修复：**

| 修复前（损坏） | 修复后 |
|-----------------|---------------|
| `std::sync::Mutex` | `tokio::sync::Mutex` |
| 在 `.await` 期间 `.lock().unwrap()` | 在 `.await` 前限定锁的作用域 |
| 获取锁无超时 | `tokio::time::timeout(dur, mutex.lock())` |
| 毒化 mutex 无恢复 | `tokio::sync::Mutex` 不会毒化 |

**预防清单：**
- [ ] 若 guard 跨越任何 `.await`，使用 `tokio::sync::Mutex`
- [ ] 为异步函数添加 `#[tracing::instrument]` 以跟踪 span
- [ ] 在预发环境运行 `tokio-console`，及早发现挂起任务
- [ ] 添加健康检查端点，验证任务响应性

<details>
<summary><strong>🏋️ 练习：找出 Bug</strong>（点击展开）</summary>

**挑战**：找出以下代码中所有异步陷阱并修复。

```rust
use std::sync::Mutex;

async fn process_requests(urls: Vec<String>) -> Vec<String> {
    let results = Mutex::new(Vec::new());
    
    for url in &urls {
        let response = reqwest::get(url).await.unwrap().text().await.unwrap();
        std::thread::sleep(std::time::Duration::from_millis(100)); // Rate limit
        let mut guard = results.lock().unwrap();
        guard.push(response);
        expensive_parse(&guard).await; // Parse all results so far
    }
    
    results.into_inner().unwrap()
}
```

<details>
<summary>🔑 解答</summary>

**发现的 Bug：**

1. **顺序抓取**——URL 逐个抓取而非并发
2. **`std::thread::sleep`**——阻塞执行器线程
3. **在 `.await` 期间持有 MutexGuard**——`guard` 在 `expensive_parse` 被 await 时仍存活
4. **无并发**——应使用 `join!` 或 `FuturesUnordered`

```rust
use tokio::sync::Mutex;
use std::sync::Arc;
use futures::stream::{self, StreamExt};

async fn process_requests(urls: Vec<String>) -> Vec<String> {
    // Fix 4: Process URLs concurrently with buffer_unordered
    let results: Vec<String> = stream::iter(urls)
        .map(|url| async move {
            let response = reqwest::get(&url).await.unwrap().text().await.unwrap();
            // Fix 2: Use tokio::time::sleep instead of std::thread::sleep
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
            response
        })
        .buffer_unordered(10) // Up to 10 concurrent requests
        .collect()
        .await;

    // Fix 3: Parse after collecting — no mutex needed at all!
    for result in &results {
        expensive_parse(result).await;
    }

    results
}
```

**要点**：通常可以重构异步代码以完全消除 mutex。用 stream/join 收集结果，再处理。更简单、更快、无死锁风险。

</details>
</details>

---

### 调试异步代码

异步堆栈跟踪出了名的晦涩——显示的是执行器的 poll 循环，而非你的逻辑调用链。以下是必备调试工具。

#### tokio-console：实时任务检查器

[tokio-console](https://github.com/tokio-rs/console) 提供类似 `htop` 的视图，展示每个已 spawn 的任务：状态、poll 时长、waker 活动与资源使用。

```toml
# Cargo.toml
[dependencies]
console-subscriber = "0.4"
tokio = { version = "1", features = ["full", "tracing"] }
```

```rust
#[tokio::main]
async fn main() {
    console_subscriber::init(); // Replaces the default tracing subscriber
    // ... rest of your application
}
```

然后在另一个终端：

```bash
$ RUSTFLAGS="--cfg tokio_unstable" cargo run   # Required compile-time flag
$ tokio-console                                # Connects to 127.0.0.1:6669
```

#### tracing + #[instrument]：异步的结构化日志

[`tracing`](https://docs.rs/tracing) crate 理解 `Future` 生命周期（lifetime）。Span 在 `.await` 点保持打开，即使 OS 线程已切换，也能给出逻辑调用栈：

```rust
use tracing::{info, instrument};

#[instrument(skip(db_pool), fields(user_id = %user_id))]
async fn handle_request(user_id: u64, db_pool: &Pool) -> Result<Response> {
    info!("looking up user");
    let user = db_pool.get_user(user_id).await?;  // span stays open across .await
    info!(email = %user.email, "found user");
    let orders = fetch_orders(user_id).await?;     // still the same span
    Ok(build_response(user, orders))
}
```

输出（使用 `tracing_subscriber::fmt::json()`）：

```json
{"timestamp":"...","level":"INFO","span":{"name":"handle_request","user_id":"42"},"message":"looking up user"}
{"timestamp":"...","level":"INFO","span":{"name":"handle_request","user_id":"42"},"fields":{"email":"a@b.com"},"message":"found user"}
```

#### 调试清单

| 症状 | 可能原因 | 工具 |
|---------|-------------|------|
| 任务永远挂起 | 缺少 `.await` 或 `Mutex` 死锁 | `tokio-console` 任务视图 |
| 吞吐量低 | 在异步线程上阻塞调用 | `tokio-console` poll 时间直方图 |
| `Future is not Send` | 在 `.await` 期间持有非 `Send` 类型 | 编译器错误 + `#[instrument]` 定位 |
| 神秘取消 | 父级 `select!` 丢弃了分支 | `tracing` span 生命周期事件 |

> **提示**：启用 `RUSTFLAGS="--cfg tokio_unstable"` 以在 tokio-console 中获取任务级指标。这是编译期标志，非运行时标志。

### 测试异步代码

异步代码带来独特的测试挑战——你需要运行时、时间控制，以及测试并发行为的策略。

**基础异步测试** 使用 `#[tokio::test]`：

```rust
// Cargo.toml
// [dev-dependencies]
// tokio = { version = "1", features = ["full", "test-util"] }

#[tokio::test]
async fn test_basic_async() {
    let result = fetch_data().await;
    assert_eq!(result, "expected");
}

// Single-threaded test (useful for !Send types):
#[tokio::test(flavor = "current_thread")]
async fn test_single_threaded() {
    let rc = std::rc::Rc::new(42);
    let val = async { *rc }.await;
    assert_eq!(val, 42);
}

// Multi-threaded with explicit worker count:
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_concurrent_behavior() {
    // Tests race conditions with real concurrency
    let counter = std::sync::Arc::new(std::sync::atomic::AtomicU32::new(0));
    let c1 = counter.clone();
    let c2 = counter.clone();
    let (a, b) = tokio::join!(
        tokio::spawn(async move { c1.fetch_add(1, std::sync::atomic::Ordering::SeqCst) }),
        tokio::spawn(async move { c2.fetch_add(1, std::sync::atomic::Ordering::SeqCst) }),
    );
    a.unwrap();
    b.unwrap();
    assert_eq!(counter.load(std::sync::atomic::Ordering::SeqCst), 2);
}
```

**时间操控**——测试超时而无需真实等待：

```rust
use tokio::time::{self, Duration, Instant};

#[tokio::test]
async fn test_timeout_behavior() {
    // Pause time — sleep() advances instantly, no real wall-clock delay
    time::pause();

    let start = Instant::now();
    time::sleep(Duration::from_secs(3600)).await; // "waits" 1 hour — takes 0ms
    assert!(start.elapsed() >= Duration::from_secs(3600));
    // Test ran in milliseconds, not an hour!
}

#[tokio::test]
async fn test_retry_timing() {
    time::pause();

    // Test that our retry logic waits the expected durations
    let start = Instant::now();
    let result = retry_with_backoff(|| async {
        Err::<(), _>("simulated failure")
    }, 3, Duration::from_secs(1))
    .await;

    assert!(result.is_err());
    // 1s + 2s + 4s = 7s of backoff (exponential)
    assert!(start.elapsed() >= Duration::from_secs(7));
}

#[tokio::test]
async fn test_deadline_exceeded() {
    time::pause();

    let result = tokio::time::timeout(
        Duration::from_secs(5),
        async {
            // Simulate slow operation
            time::sleep(Duration::from_secs(10)).await;
            "done"
        }
    ).await;

    assert!(result.is_err()); // Timed out
}
```

**Mock 异步依赖**——使用 trait 对象或泛型：

```rust
// Define a trait for the dependency:
trait Storage {
    async fn get(&self, key: &str) -> Option<String>;
    async fn set(&self, key: &str, value: String);
}

// Production implementation:
struct RedisStorage { /* ... */ }
impl Storage for RedisStorage {
    async fn get(&self, key: &str) -> Option<String> {
        // Real Redis call
        todo!()
    }
    async fn set(&self, key: &str, value: String) {
        todo!()
    }
}

// Test mock:
struct MockStorage {
    data: std::sync::Mutex<std::collections::HashMap<String, String>>,
}

impl MockStorage {
    fn new() -> Self {
        MockStorage { data: std::sync::Mutex::new(std::collections::HashMap::new()) }
    }
}

impl Storage for MockStorage {
    async fn get(&self, key: &str) -> Option<String> {
        self.data.lock().unwrap().get(key).cloned()
    }
    async fn set(&self, key: &str, value: String) {
        self.data.lock().unwrap().insert(key.to_string(), value);
    }
}

// Tested function is generic over Storage:
async fn cache_lookup<S: Storage>(store: &S, key: &str) -> String {
    match store.get(key).await {
        Some(val) => val,
        None => {
            let val = "computed".to_string();
            store.set(key, val.clone()).await;
            val
        }
    }
}

#[tokio::test]
async fn test_cache_miss_then_hit() {
    let mock = MockStorage::new();

    // First call: miss → computes and stores
    let val = cache_lookup(&mock, "key1").await;
    assert_eq!(val, "computed");

    // Second call: hit → returns stored value
    let val = cache_lookup(&mock, "key1").await;
    assert_eq!(val, "computed");
    assert!(mock.data.lock().unwrap().contains_key("key1"));
}
```

**测试 channel 与任务通信**：

```rust
#[tokio::test]
async fn test_producer_consumer() {
    let (tx, mut rx) = tokio::sync::mpsc::channel(10);

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
        // tx dropped here — channel closes
    });

    let mut received = Vec::new();
    while let Some(val) = rx.recv().await {
        received.push(val);
    }

    assert_eq!(received, vec![0, 1, 2, 3, 4]);
}
```

| 测试模式 | 适用场景 | 关键工具 |
|-------------|-------------|----------|
| `#[tokio::test]` | 所有异步测试 | `tokio = { features = ["macros", "rt"] }` |
| `time::pause()` | 测试超时、重试、周期性任务 | `tokio::time::pause()` |
| Trait mock | 无 I/O 测试业务逻辑 | 泛型 `<S: Storage>` |
| `current_thread` flavor | 测试 `!Send` 类型或确定性调度 | `#[tokio::test(flavor = "current_thread")]` |
| `multi_thread` flavor | 测试竞态条件 | `#[tokio::test(flavor = "multi_thread")]` |

> **要点回顾 — 常见陷阱**
> - 永远不要阻塞执行器——CPU/同步工作使用 `spawn_blocking`
> - 永远不要在 `.await` 期间持有 `MutexGuard`——严格限定锁作用域或使用 `tokio::sync::Mutex`
> - 取消会立即丢弃 Future——对部分操作使用「可安全取消」模式
> - 使用 `tokio-console` 和 `#[tracing::instrument]` 调试异步代码
> - 用 `#[tokio::test]` 和 `time::pause()` 进行确定性时序测试

> **另见：** [第 8 章 — Tokio 深入](ch08-tokio-deep-dive.md) 了解同步原语，[第 13 章 — 生产模式](ch13-production-patterns.md) 了解优雅关闭与结构化并发

***

