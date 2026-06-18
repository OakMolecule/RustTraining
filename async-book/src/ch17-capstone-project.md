# 毕业项目：异步聊天服务器

本项目将全书模式整合为单一的生产风格应用。你将使用 tokio、channel、stream、优雅关闭和正确的错误处理，构建一个 **多房间异步聊天服务器**。

**预计时间**：4–6 小时 | **难度**：★★★

> **你将练习：**
> - `tokio::spawn` 与 `'static` 要求（第 8 章）
> - Channel：消息用 `mpsc`、房间用 `broadcast`、关闭用 `watch`（第 8 章）
> - Stream：从 TCP 连接读取行（第 11 章）
> - 常见陷阱：取消安全、在 `.await` 期间持有 MutexGuard（第 12 章）
> - 生产模式：优雅关闭、背压（第 13 章）
> - 可插拔后端的异步 Trait（第 10 章）

## 问题描述

构建 TCP 聊天服务器，要求：

1. **客户端** 通过 TCP 连接并加入命名房间
2. **消息** 广播给同一房间的所有客户端
3. **命令**：`/join <room>`、`/nick <name>`、`/rooms`、`/quit`
4. 服务器在 Ctrl+C 时优雅关闭——完成进行中的消息

```mermaid
graph LR
    C1["客户端 1<br/>(Alice)"] -->|TCP| SERVER["聊天服务器"]
    C2["客户端 2<br/>(Bob)"] -->|TCP| SERVER
    C3["客户端 3<br/>(Carol)"] -->|TCP| SERVER

    SERVER --> R1["#general<br/>broadcast channel"]
    SERVER --> R2["#rust<br/>broadcast channel"]

    R1 -->|msg| C1
    R1 -->|msg| C2
    R2 -->|msg| C3

    CTRL["Ctrl+C"] -->|watch| SERVER

    style SERVER fill:#e8f4f8,stroke:#2980b9,color:#000
    style R1 fill:#d4efdf,stroke:#27ae60,color:#000
    style R2 fill:#d4efdf,stroke:#27ae60,color:#000
    style CTRL fill:#fadbd8,stroke:#e74c3c,color:#000
```

## 步骤 1：基础 TCP Accept 循环

从接受连接并回显行的服务器开始：

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Chat server listening on :8080");

    loop {
        let (socket, addr) = listener.accept().await?;
        println!("[{addr}] Connected");

        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut reader = BufReader::new(reader);
            let mut line = String::new();

            loop {
                line.clear();
                match reader.read_line(&mut line).await {
                    Ok(0) | Err(_) => break,
                    Ok(_) => {
                        let _ = writer.write_all(line.as_bytes()).await;
                    }
                }
            }
            println!("[{addr}] Disconnected");
        });
    }
}
```

**你的任务**：验证可编译，并用 `telnet localhost 8080` 测试。

## 步骤 2：用 Broadcast Channel 管理房间状态

每个房间是一个 `broadcast::Sender`。房间内所有客户端订阅以接收消息。

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};

type RoomMap = Arc<RwLock<HashMap<String, broadcast::Sender<String>>>>;

fn get_or_create_room(rooms: &mut HashMap<String, broadcast::Sender<String>>, name: &str) -> broadcast::Sender<String> {
    rooms.entry(name.to_string())
        .or_insert_with(|| {
            let (tx, _) = broadcast::channel(100); // 100-message buffer
            tx
        })
        .clone()
}
```

**你的任务**：实现房间状态，使得：
- 客户端默认在 `#general`
- `/join <room>` 切换房间（取消订阅旧房间，订阅新房间）
- 消息广播给发送者当前房间的所有客户端

<details>
<summary>💡 提示 — 客户端任务结构</summary>

每个客户端任务需要两个并发循环：
1. **从 TCP 读取** → 解析命令或向房间广播
2. **从 broadcast receiver 读取** → 写入 TCP

使用 `tokio::select!` 同时运行两者：

```rust
loop {
    tokio::select! {
        // Client sent us a line
        result = reader.read_line(&mut line) => {
            match result {
                Ok(0) | Err(_) => break,
                Ok(_) => {
                    // Parse command or broadcast message
                }
            }
        }
        // Room broadcast received
        result = room_rx.recv() => {
            match result {
                Ok(msg) => {
                    let _ = writer.write_all(msg.as_bytes()).await;
                }
                Err(_) => break,
            }
        }
    }
}
```

</details>

## 步骤 3：命令

实现命令协议：

| 命令 | 动作 |
|---------|--------|
| `/join <room>` | 离开当前房间，加入新房间，在两个房间中公告 |
| `/nick <name>` | 更改显示名称 |
| `/rooms` | 列出所有活跃房间及成员数 |
| `/quit` | 优雅断开 |
| 其他内容 | 作为聊天消息广播 |

**你的任务**：从输入行解析命令。对于 `/rooms`，需从 `RoomMap` 读取——使用 `RwLock::read()` 避免阻塞其他客户端。

## 步骤 4：优雅关闭

添加 Ctrl+C 处理，使服务器：
1. 停止接受新连接
2. 向所有房间发送「Server shutting down...」
3. 等待进行中的消息排空
4. 干净退出

```rust
use tokio::sync::watch;

let (shutdown_tx, shutdown_rx) = watch::channel(false);

// In the accept loop:
loop {
    tokio::select! {
        result = listener.accept() => {
            let (socket, addr) = result?;
            // spawn client task with shutdown_rx.clone()
        }
        _ = tokio::signal::ctrl_c() => {
            println!("Shutdown signal received");
            shutdown_tx.send(true)?;
            break;
        }
    }
}
```

**你的任务**：在每个客户端的 `select!` 循环中添加 `shutdown_rx.changed()`，以便收到关闭信号时客户端退出。

## 步骤 5：错误处理与边界情况

生产级加固服务器：

1. **滞后接收者（lagging receivers）**：若慢客户端错过消息，`broadcast::recv()` 返回 `RecvError::Lagged(n)`。优雅处理（记录日志并继续，不要崩溃）。
2. **昵称验证**：拒绝空或过长昵称。
3. **背压**：broadcast channel 缓冲区有界（100）。若客户端跟不上，会收到 `Lagged` 错误。
4. **超时**：空闲超过 5 分钟的客户端断开连接。

```rust
use tokio::time::{timeout, Duration};

// Wrap the read in a timeout:
match timeout(Duration::from_secs(300), reader.read_line(&mut line)).await {
    Ok(Ok(0)) | Ok(Err(_)) | Err(_) => break, // EOF, error, or timeout
    Ok(Ok(_)) => { /* process line */ }
}
```

## 步骤 6：集成测试

编写测试：启动服务器、连接两个客户端、验证消息投递：

```rust
#[tokio::test]
async fn two_clients_can_chat() {
    // Start server in background
    let server = tokio::spawn(run_server("127.0.0.1:0")); // Port 0 = OS picks

    // Connect two clients
    let mut client1 = TcpStream::connect(addr).await.unwrap();
    let mut client2 = TcpStream::connect(addr).await.unwrap();

    // Client 1 sends a message
    client1.write_all(b"Hello from client 1\n").await.unwrap();

    // Client 2 should receive it
    let mut buf = vec![0u8; 1024];
    let n = client2.read(&mut buf).await.unwrap();
    let msg = String::from_utf8_lossy(&buf[..n]);
    assert!(msg.contains("Hello from client 1"));
}
```

## 评估标准

| 标准 | 目标 |
|-----------|--------|
| 并发 | 多房间多客户端，无阻塞 |
| 正确性 | 消息仅发送给同房间客户端 |
| 优雅关闭 | Ctrl+C 排空消息并干净退出 |
| 错误处理 | 处理滞后接收者、断开、超时 |
| 代码组织 | 清晰分离：accept 循环、客户端任务、房间状态 |
| 测试 | 至少 2 个集成测试 |

## 扩展想法

基础聊天服务器完成后，可尝试：

1. **持久历史**：每房间存储最近 N 条消息；新加入者回放
2. **WebSocket 支持**：使用 `tokio-tungstenite` 同时接受 TCP 和 WebSocket 客户端
3. **限流**：使用 `tokio::time::Interval` 限制每客户端每秒消息数
4. **指标**：通过 `prometheus` crate 跟踪连接客户端数、消息/秒、房间数
5. **TLS**：添加 `tokio-rustls` 实现加密连接

***

