# 总结与参考卡

## 快速参考卡

### 异步心智模型

```text
┌─────────────────────────────────────────────────────┐
│  async fn → State Machine (enum) → impl Future     │
│  .await   → poll() the inner future                 │
│  executor → loop { poll(); sleep_until_woken(); }   │
│  waker    → "hey executor, poll me again"           │
│  Pin      → "promise I won't move in memory"        │
└─────────────────────────────────────────────────────┘
```

### 常见模式速查表

| 目标 | 使用 |
|------|-----|
| 并发运行两个 Future | `tokio::join!(a, b)` |
| 竞速两个 Future | `tokio::select! { ... }` |
| spawn 后台任务 | `tokio::spawn(async { ... })` |
| 在异步中运行阻塞代码 | `tokio::task::spawn_blocking(\|\| { ... })` |
| 限制并发 | `Semaphore::new(N)` |
| 收集多个任务结果 | `JoinSet` |
| 在任务间共享状态 | `Arc<Mutex<T>>` 或 channel |
| 优雅关闭 | `watch::channel` + `select!` |
| 每次处理 N 个 Stream 元素 | `.buffer_unordered(N)` |
| 为 Future 设置超时 | `tokio::time::timeout(dur, fut)` |
| 带退避的重试 | 自定义组合子（见第 13 章） |

### Pin 快速参考

| 场景 | 使用 |
|-----------|-----|
| 在堆上 Pin Future | `Box::pin(fut)` |
| 在栈上 Pin Future | `tokio::pin!(fut)` |
| Pin `Unpin` 类型 | `Pin::new(&mut val)` — 安全、无成本 |
| 返回 Pin 的 trait 对象 | `-> Pin<Box<dyn Future<Output = T> + Send>>` |

### Channel 选择指南

| Channel | 生产者 | 消费者 | 值 | 适用场景 |
|---------|-----------|-----------|--------|----------|
| `mpsc` | N | 1 | Stream | 工作队列、事件总线 |
| `oneshot` | 1 | 1 | 单个 | 请求/响应、完成通知 |
| `broadcast` | N | N | 全部接收 | 扇出通知、关闭信号 |
| `watch` | 1 | N | 仅最新 | 配置更新、健康状态 |

### Mutex 选择指南

| Mutex | 适用场景 |
|-------|----------|
| `std::sync::Mutex` | 短暂持有锁，永不跨越 `.await` |
| `tokio::sync::Mutex` | 必须在 `.await` 期间持有锁 |
| `parking_lot::Mutex` | 高争用、无 `.await`、需要性能 |
| `tokio::sync::RwLock` | 多读少写，锁跨越 `.await` |

### 决策快速参考

```text
需要并发？
├── I/O 密集型 → async/await
├── CPU 密集型 → rayon / std::thread
└── 混合型 → CPU 部分用 spawn_blocking

选择运行时？
├── 服务器应用 → tokio
├── 库 → 与运行时无关（futures crate）
├── 嵌入式 → embassy
└── 极简 → smol

需要并发 Future？
├── 可以是 'static + Send → tokio::spawn
├── 可以是 'static + !Send → LocalSet
├── 不能是 'static → FuturesUnordered
└── 需要跟踪/中止 → JoinSet
```

### 常见错误信息与修复

| 错误 | 原因 | 修复 |
|-------|-------|-----|
| `future is not Send` | 在 `.await` 期间持有 `!Send` 类型 | 在 `.await` 前限定值的作用域使其被丢弃，或使用 `current_thread` 运行时 |
| spawn 中 `borrowed value does not live long enough` | `tokio::spawn` 需要 `'static` | 使用 `Arc`、`clone()` 或 `FuturesUnordered` |
| `the trait Future is not implemented for ()` | 缺少 `.await` | 为异步调用添加 `.await` |
| poll 中 `cannot borrow as mutable` | 自引用借用 | 正确使用 `Pin<&mut Self>`（见第 4 章） |
| 程序静默挂起 | 忘记调用 `waker.wake()` | 确保每个 `Pending` 路径注册并触发 waker |

### 延伸阅读

| 资源 | 原因 |
|----------|-----|
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | 官方动手教程——非常适合首个项目 |
| [Async Book (official)](https://rust-lang.github.io/async-book/) | 在语言层面讲解 `Future`、`Pin`、`Stream` |
| [Jon Gjengset — Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM) | 2 小时深入内部机制与现场编码 |
| [Alice Ryhl — Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | 有状态服务的生产架构模式 |
| [Without Boats — Pin, Unpin, and why Rust needs them](https://without.boats.blog/pin/) | 语言设计者的原始动机 |
| [Tokio mini-Redis](https://github.com/tokio-rs/mini-redis) | 完整异步 Rust 项目——值得研读的生产代码 |
| [Tower documentation](https://docs.rs/tower) | axum、tonic、hyper 使用的中间件/服务架构 |

***

*异步 Rust 培训指南 完*
