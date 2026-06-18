# 11. Stream 与 AsyncIterator 🟡

> **你将学到：**
> - `Stream` Trait（特征）：异步迭代多个值
> - 创建 Stream：`stream::iter`、`async_stream`、`unfold`
> - Stream 组合子：`map`、`filter`、`buffer_unordered`、`fold`
> - 异步 I/O Trait：`AsyncRead`、`AsyncWrite`、`AsyncBufRead`

## Stream Trait 概览

`Stream` 之于 `Iterator`，就像 `Future` 之于单个值——它异步地产生多个值：

```rust
// std::iter::Iterator (synchronous, multiple values)
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// futures::Stream (async, multiple values)
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

```mermaid
graph LR
    subgraph "同步"
        VAL["值<br/>(T)"]
        ITER["迭代器<br/>(多个 T)"]
    end

    subgraph "异步"
        FUT["Future<br/>(异步 T)"]
        STREAM["Stream<br/>(异步多个 T)"]
    end

    VAL -->|"变为异步"| FUT
    ITER -->|"变为异步"| STREAM
    VAL -->|"变为多个"| ITER
    FUT -->|"变为多个"| STREAM

    style VAL fill:#e3f2fd,color:#000
    style ITER fill:#e3f2fd,color:#000
    style FUT fill:#c8e6c9,color:#000
    style STREAM fill:#c8e6c9,color:#000
```

### 创建 Stream

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{interval, Duration};
use tokio_stream::wrappers::IntervalStream;

// 1. From an iterator
let s = stream::iter(vec![1, 2, 3]);

// 2. From an async generator (using async_stream crate)
// Cargo.toml: async-stream = "0.3"
use async_stream::stream;

fn countdown(from: u32) -> impl futures::Stream<Item = u32> {
    stream! {
        for i in (0..=from).rev() {
            tokio::time::sleep(Duration::from_millis(500)).await;
            yield i;
        }
    }
}

// 3. From a tokio interval
let tick_stream = IntervalStream::new(interval(Duration::from_secs(1)));

// 4. From a channel receiver (tokio_stream::wrappers)
let (tx, rx) = tokio::sync::mpsc::channel::<String>(100);
let rx_stream = tokio_stream::wrappers::ReceiverStream::new(rx);

// 5. From unfold (generate from async state)
let s = stream::unfold(0u32, |state| async move {
    if state >= 5 {
        None // Stream ends
    } else {
        let next = state + 1;
        Some((state, next)) // yield `state`, new state is `next`
    }
});
```

### 消费 Stream

```rust
use futures::stream::{self, StreamExt};

async fn stream_examples() {
    let s = stream::iter(vec![1, 2, 3, 4, 5]);

    // for_each — process each item
    s.for_each(|x| async move {
        println!("{x}");
    }).await;

    // map + collect
    let doubled: Vec<i32> = stream::iter(vec![1, 2, 3])
        .map(|x| x * 2)
        .collect()
        .await;

    // filter
    let evens: Vec<i32> = stream::iter(1..=10)
        .filter(|x| futures::future::ready(x % 2 == 0))
        .collect()
        .await;

    // buffer_unordered — process N items concurrently
    let results: Vec<_> = stream::iter(vec!["url1", "url2", "url3"])
        .map(|url| async move {
            // Simulate HTTP fetch
            tokio::time::sleep(Duration::from_millis(100)).await;
            format!("response from {url}")
        })
        .buffer_unordered(10) // Up to 10 concurrent fetches
        .collect()
        .await;

    // take, skip, zip, chain — just like Iterator
    let first_three: Vec<i32> = stream::iter(1..=100)
        .take(3)
        .collect()
        .await;
}
```

### 与 C# IAsyncEnumerable 对比

| 特性 | Rust `Stream` | C# `IAsyncEnumerable<T>` |
|---------|--------------|--------------------------|
| **语法** | `stream! { yield x; }` | `await foreach` / `yield return` |
| **取消** | 丢弃 Stream | `CancellationToken` |
| **背压（backpressure）** | 消费者控制 poll 速率 | 消费者控制 `MoveNextAsync` |
| **内置** | 否（需要 `futures` crate） | 是（自 C# 8.0 起） |
| **组合子** | `.map()`、`.filter()`、`.buffer_unordered()` | LINQ + `System.Linq.Async` |
| **错误处理** | `Stream<Item = Result<T, E>>` | 在异步迭代器中抛出异常 |

```rust
// Rust: Stream of database rows
// NOTE: try_stream! (not stream!) is required when using ? inside the body.
// stream! doesn't propagate errors — try_stream! yields Err(e) and ends.
fn get_users(db: &Database) -> impl Stream<Item = Result<User, DbError>> + '_ {
    try_stream! {
        let mut cursor = db.query("SELECT * FROM users").await?;
        while let Some(row) = cursor.next().await {
            yield User::from_row(row?);
        }
    }
}

// Consume:
let mut users = pin!(get_users(&db));
while let Some(result) = users.next().await {
    match result {
        Ok(user) => println!("{}", user.name),
        Err(e) => eprintln!("Error: {e}"),
    }
}
```

```csharp
// C# equivalent:
async IAsyncEnumerable<User> GetUsers() {
    await using var reader = await db.QueryAsync("SELECT * FROM users");
    while (await reader.ReadAsync()) {
        yield return User.FromRow(reader);
    }
}

// Consume:
await foreach (var user in GetUsers()) {
    Console.WriteLine(user.Name);
}
```

<details>
<summary><strong>🏋️ 练习：构建异步统计聚合器</strong>（点击展开）</summary>

**挑战**：给定一个传感器读数 Stream `Stream<Item = f64>`，编写一个异步函数，消费该 Stream 并返回 `(count, min, max, average)`。使用 `StreamExt` 组合子——不要只是收集到 `Vec` 中。

*提示*：使用 `.fold()` 在 Stream 上累积状态。

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{self, StreamExt};

#[derive(Debug)]
struct Stats {
    count: usize,
    min: f64,
    max: f64,
    sum: f64,
}

impl Stats {
    fn average(&self) -> f64 {
        if self.count == 0 { 0.0 } else { self.sum / self.count as f64 }
    }
}

async fn compute_stats<S: futures::Stream<Item = f64>>(stream: S) -> Stats {
    stream
        .fold(
            Stats { count: 0, min: f64::INFINITY, max: f64::NEG_INFINITY, sum: 0.0 },
            |mut acc, value| async move {
                acc.count += 1;
                acc.min = acc.min.min(value);
                acc.max = acc.max.max(value);
                acc.sum += value;
                acc
            },
        )
        .await
}

#[tokio::test]
async fn test_stats() {
    let readings = stream::iter(vec![23.5, 24.1, 22.8, 25.0, 23.9]);
    let stats = compute_stats(readings).await;

    assert_eq!(stats.count, 5);
    assert!((stats.min - 22.8).abs() < f64::EPSILON);
    assert!((stats.max - 25.0).abs() < f64::EPSILON);
    assert!((stats.average() - 23.86).abs() < 0.01);
}
```

**要点**：像 `.fold()` 这样的 Stream 组合子可以逐条处理元素，而无需全部载入内存——这对处理大型或无界数据流至关重要。

</details>
</details>

### 异步 I/O Trait：AsyncRead、AsyncWrite、AsyncBufRead

正如 `std::io::Read`/`Write` 是同步 I/O 的基础，它们的异步对应物是异步 I/O 的基础。这些 Trait 由 `tokio::io` 提供（或与运行时无关的代码使用 `futures::io`）：

```rust
// tokio::io — the async versions of std::io traits

/// Read bytes from a source asynchronously
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,  // Tokio's safe wrapper around uninitialized memory
    ) -> Poll<io::Result<()>>;
}

/// Write bytes to a sink asynchronously
pub trait AsyncWrite {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<io::Result<usize>>;

    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
    fn poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
}

/// Buffered reading with line support
pub trait AsyncBufRead: AsyncRead {
    fn poll_fill_buf(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<&[u8]>>;
    fn consume(self: Pin<&mut Self>, amt: usize);
}
```

**实践中**，你很少直接调用这些 `poll_*` 方法。相反，应使用扩展 Trait `AsyncReadExt` 和 `AsyncWriteExt`，它们提供支持 `.await` 的辅助方法：

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt};
use tokio::net::TcpStream;
use tokio::io::BufReader;

async fn io_examples() -> tokio::io::Result<()> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    // AsyncWriteExt: write_all, write_u32, write_buf, etc.
    stream.write_all(b"GET / HTTP/1.0\r\n\r\n").await?;

    // AsyncReadExt: read, read_exact, read_to_end, read_to_string
    let mut response = Vec::new();
    stream.read_to_end(&mut response).await?;

    // AsyncBufReadExt: read_line, lines(), split()
    let file = tokio::fs::File::open("config.txt").await?;
    let reader = BufReader::new(file);
    let mut lines = reader.lines();
    while let Some(line) = lines.next_line().await? {
        println!("{line}");
    }

    Ok(())
}
```

**实现自定义异步 I/O**——在原始 TCP 之上封装协议：

```rust
use tokio::io::{AsyncRead, AsyncWrite, ReadBuf};
use std::pin::Pin;
use std::task::{Context, Poll};

/// A length-prefixed protocol: [u32 length][payload bytes]
struct FramedStream<T> {
    inner: T,
}

impl<T: AsyncRead + AsyncReadExt + Unpin> FramedStream<T> {
    /// Read one complete frame
    async fn read_frame(&mut self) -> tokio::io::Result<Vec<u8>>
    {
        // Read the 4-byte length prefix
        let len = self.inner.read_u32().await? as usize;

        // Read exactly that many bytes
        let mut payload = vec![0u8; len];
        self.inner.read_exact(&mut payload).await?;
        Ok(payload)
    }
}

impl<T: AsyncWrite + AsyncWriteExt + Unpin> FramedStream<T> {
    /// Write one complete frame
    async fn write_frame(&mut self, data: &[u8]) -> tokio::io::Result<()>
    {
        self.inner.write_u32(data.len() as u32).await?;
        self.inner.write_all(data).await?;
        self.inner.flush().await?;
        Ok(())
    }
}
```

| 同步 Trait | 异步 Trait (tokio) | 异步 Trait (futures) | 扩展 Trait |
|-----------|--------------------|-----------------------|----------------|
| `std::io::Read` | `tokio::io::AsyncRead` | `futures::io::AsyncRead` | `AsyncReadExt` |
| `std::io::Write` | `tokio::io::AsyncWrite` | `futures::io::AsyncWrite` | `AsyncWriteExt` |
| `std::io::BufRead` | `tokio::io::AsyncBufRead` | `futures::io::AsyncBufRead` | `AsyncBufReadExt` |
| `std::io::Seek` | `tokio::io::AsyncSeek` | `futures::io::AsyncSeek` | `AsyncSeekExt` |

> **tokio 与 futures I/O Trait 对比**：它们相似但不完全相同——tokio 的 `AsyncRead` 使用 `ReadBuf`（安全处理未初始化内存），而 `futures::AsyncRead` 使用 `&mut [u8]`。使用 `tokio_util::compat` 在两者之间转换。

> **复制工具**：`tokio::io::copy(&mut reader, &mut writer)` 是 `std::io::copy` 的异步等价物——适用于代理服务器或文件传输。`tokio::io::copy_bidirectional` 并发地向两个方向复制。

<details>
<summary><strong>🏋️ 练习：构建异步行计数器</strong>（点击展开）</summary>

**挑战**：编写一个异步函数，接受任意 `AsyncBufRead` 数据源并返回非空行的数量。它应适用于文件、TCP 流或任何带缓冲的 reader。

*提示*：使用 `AsyncBufReadExt::lines()`，并统计 `!line.is_empty()` 的行。

<details>
<summary>🔑 解答</summary>

```rust
use tokio::io::AsyncBufReadExt;

async fn count_non_empty_lines<R: tokio::io::AsyncBufRead + Unpin>(
    reader: R,
) -> tokio::io::Result<usize> {
    let mut lines = reader.lines();
    let mut count = 0;
    while let Some(line) = lines.next_line().await? {
        if !line.is_empty() {
            count += 1;
        }
    }
    Ok(count)
}

// Works with any AsyncBufRead:
// let file = tokio::io::BufReader::new(tokio::fs::File::open("data.txt").await?);
// let count = count_non_empty_lines(file).await?;
//
// let tcp = tokio::io::BufReader::new(TcpStream::connect("...").await?);
// let count = count_non_empty_lines(tcp).await?;
```

**要点**：针对 `AsyncBufRead` 而非具体类型编程，你的 I/O 代码可在文件、套接字、管道之间复用，甚至适用于内存缓冲区（`tokio::io::BufReader::new(std::io::Cursor::new(data))`）。

</details>
</details>

> **要点回顾 — Stream 与 AsyncIterator**
> - `Stream` 是 `Iterator` 的异步等价物——产生 `Poll::Ready(Some(item))` 或 `Poll::Ready(None)`
> - `.buffer_unordered(N)` 并发处理 N 个 Stream 元素——Stream 的关键并发工具
> - `async_stream::stream!` 是创建自定义 Stream 的最简单方式（使用 `yield`）
> - `AsyncRead`/`AsyncBufRead` 使 I/O 代码可在文件、套接字和管道之间通用复用

> **另见：** [第 9 章 — 何时 Tokio 并非合适选择](ch09-when-tokio-isnt-the-right-fit.md) 了解 `FuturesUnordered`（相关模式），[第 13 章 — 生产模式](ch13-production-patterns.md) 了解有界 channel 的背压

***

