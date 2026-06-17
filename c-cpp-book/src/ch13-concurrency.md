# Rust 并发 {#rust-concurrency}

> **你将学到：** Rust 的并发模型——线程、`Send`/`Sync` 标记 Trait、`Mutex<T>`、`Arc<T>`、通道，以及编译器如何在编译期防止数据竞争。未使用的线程安全无运行时开销。

- Rust 内置并发支持，类似 C++ 中的 `std::thread`
    - 关键差异：Rust 通过 `Send` 和 `Sync` 标记 Trait **在编译期防止数据竞争**
    - 在 C++ 中，在没有 mutex 的情况下跨线程共享 `std::vector` 是 UB 但能编译通过。在 Rust 中，无法编译。
    - Rust 中的 `Mutex<T>` 包装的是**数据**，而不只是访问——不锁定就无法读取数据
- `thread::spawn()` 可用于创建单独线程，并行执行闭包 `||`
```rust
use std::thread;
use std::time::Duration;
fn main() {
    let handle = thread::spawn(|| {
        for i in 0..10 {
            println!("Count in thread: {i}!");
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 0..5 {
        println!("Main thread: {i}");
        thread::sleep(Duration::from_millis(5));
    }

    handle.join().unwrap(); // The handle.join() ensures that the spawned thread exits
}
```

# Rust 并发
- 当需要从环境借用时，可使用 ```thread::scope()```。这可行是因为 ```thread::scope``` 会等待内部线程返回
- 尝试不使用 ```thread::scope``` 执行此练习以了解问题
```rust
use std::thread;
fn main() {
  let a = [0, 1, 2];
  thread::scope(|scope| {
      scope.spawn(|| {
          for x in &a {
            println!("{x}");
          }
      });
  });
}
```
----
# Rust 并发
- 也可使用 ```move``` 将所有权转移给线程。对于像 `[i32; 3]` 这样的 `Copy` 类型，`move` 关键字将数据复制到闭包中，原变量仍可使用
```rust
use std::thread;
fn main() {
  let mut a = [0, 1, 2];
  let handle = thread::spawn(move || {
      for x in a {
        println!("{x}");
      }
  });
  a[0] = 42;    // Doesn't affect the copy sent to the thread
  handle.join().unwrap();
}
```

# Rust 并发
- ```Arc<T>``` 可用于在多个线程间共享*只读*引用
    - ```Arc``` 表示 Atomic Reference Counted（原子引用计数）。引用直到引用计数为 0 才释放
    - ```Arc::clone()``` 仅增加引用计数，不克隆数据
```rust
use std::sync::Arc;
use std::thread;
fn main() {
    let a = Arc::new([0, 1, 2]);
    let mut handles = Vec::new();
    for i in 0..2 {
        let arc = Arc::clone(&a);
        handles.push(thread::spawn(move || {
            println!("Thread: {i} {arc:?}");
        }));
    }
    handles.into_iter().for_each(|h| h.join().unwrap());
}
```

# Rust 并发
- ```Arc<T>``` 可与 ```Mutex<T>``` 结合提供可变引用。
    - ```Mutex``` 保护受保护的数据，确保只有持有锁的线程才能访问。
    - `MutexGuard` 在超出作用域时自动释放（RAII）。注意：`std::mem::forget` 仍可能泄漏 guard——因此「不可能忘记解锁」比「不可能泄漏」更准确。
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
            // MutexGuard dropped here — lock released automatically
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final count: {}", *counter.lock().unwrap());
    // Output: Final count: 5
}
```

# Rust 并发：RwLock
- `RwLock<T>` 允许多个并发读者或**一个**独占写者——C++ 中的读写锁模式（`std::shared_mutex`）
    - 读远多于写时使用 `RwLock`（例如配置、缓存）
    - 读写频率相近或临界区较短时使用 `Mutex`
```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("v1.0")));
    let mut handles = Vec::new();

    // Spawn 5 readers — all can run concurrently
    for i in 0..5 {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let val = config.read().unwrap();  // Multiple readers OK
            println!("Reader {i}: {val}");
        }));
    }

    // One writer — blocks until all readers finish
    {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let mut val = config.write().unwrap();  // Exclusive access
            *val = String::from("v2.0");
            println!("Writer: updated to {val}");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

# Rust 并发：Mutex 中毒
- 如果线程在持有 `Mutex` 或 `RwLock` 时 **panic**，锁会变为**中毒**状态
    - 后续对 `.lock()` 的调用返回 `Err(PoisonError)`——数据可能处于不一致状态
    - 若确信数据仍有效，可用 `.into_inner()` 恢复
    - C++ 中没有等价物——`std::mutex` 没有中毒概念；panic 的线程只是让锁保持持有状态
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let data2 = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut guard = data2.lock().unwrap();
        guard.push(4);
        panic!("oops!");  // Lock is now poisoned
    });

    let _ = handle.join();  // Thread panicked

    // Subsequent lock attempts return Err(PoisonError)
    match data.lock() {
        Ok(guard) => println!("Data: {guard:?}"),
        Err(poisoned) => {
            println!("Lock was poisoned! Recovering...");
            let guard = poisoned.into_inner();  // Access data anyway
            println!("Recovered data: {guard:?}");  // [1, 2, 3, 4] — push succeeded before panic
        }
    }
}
```

# Rust 并发：原子类型
- 对于简单计数器和标志，`std::sync::atomic` 类型可避免 `Mutex` 的开销
    - `AtomicBool`、`AtomicI32`、`AtomicU64`、`AtomicUsize` 等
    - 等价于 C++ 的 `std::atomic<T>`——相同的内存序模型（`Relaxed`、`Acquire`、`Release`、`SeqCst`）
```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", counter.load(Ordering::SeqCst));
    // Output: Counter: 10000
}
```

| 原语 | 何时使用 | C++ 等价物 |
|-----------|-------------|----------------|
| `Mutex<T>` | 一般可变共享状态 | `std::mutex` + 手动关联数据 |
| `RwLock<T>` | 读多写少的工作负载 | `std::shared_mutex` |
| `Atomic*` | 简单计数器、标志、无锁模式 | `std::atomic<T>` |
| `Condvar` | 等待条件变为真 | `std::condition_variable` |

# Rust 并发：Condvar
- `Condvar`（条件变量）让线程**休眠直到另一线程发出信号**表明条件已改变
    - 始终与 `Mutex` 配对——模式是：加锁、检查条件、未就绪则等待、就绪后行动
    - 等价于 C++ 的 `std::condition_variable` / `std::condition_variable::wait`
    - 处理**虚假唤醒**——始终在循环中重新检查条件（或使用 `wait_while`/`wait_until`）
```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));

    // Spawn a worker that waits for a signal
    let pair2 = Arc::clone(&pair);
    let worker = thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut ready = lock.lock().unwrap();
        // wait: sleeps until signaled (always re-check in a loop for spurious wakeups)
        while !*ready {
            ready = cvar.wait(ready).unwrap();
        }
        println!("Worker: condition met, proceeding!");
    });

    // Main thread does some work, then signals the worker
    thread::sleep(std::time::Duration::from_millis(100));
    {
        let (lock, cvar) = &*pair;
        let mut ready = lock.lock().unwrap();
        *ready = true;
        cvar.notify_one();  // Wake one waiting thread (notify_all() wakes all)
    }

    worker.join().unwrap();
}
```

> **何时使用 Condvar 与通道：** 当线程共享可变状态并需要等待该状态上的条件时使用 `Condvar`（例如「缓冲区非空」）。当线程需要传递*消息*时使用通道（`mpsc`）。通道通常更易推理。

# Rust 并发
- Rust 通道可用于在 ```Sender``` 和 ```Receiver``` 之间交换消息
    - 这使用称为 ```mpsc``` 或 ```Multi-producer, Single-Consumer``` 的范式
    - ```send()``` 和 ```recv()``` 都可能阻塞线程
```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    tx.send(10).unwrap();
    tx.send(20).unwrap();
    
    println!("Received: {:?}", rx.recv());
    println!("Received: {:?}", rx.recv());

    let tx2 = tx.clone();
    tx2.send(30).unwrap();
    println!("Received: {:?}", rx.recv());
}
```

# Rust 并发
- 通道可与线程结合
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    for _ in 0..2 {
        let tx2 = tx.clone();
        thread::spawn(move || {
            let thread_id = thread::current().id();
            for i in 0..10 {
                tx2.send(format!("Message {i}")).unwrap();
                println!("{thread_id:?}: sent Message {i}");
            }
            println!("{thread_id:?}: done");
        });
    }

        // Drop the original sender so rx.iter() terminates when all cloned senders are dropped
    drop(tx);

    thread::sleep(Duration::from_millis(100));

    for msg in rx.iter() {
        println!("Main: got {msg}");
    }
}
```



## Rust 为何能防止数据竞争：Send 与 Sync {#why-rust-prevents-data-races-send-and-sync}

- Rust 使用两个标记 Trait 在编译期强制线程安全：
    - `Send`：若类型可安全**转移**到另一线程，则该类型是 `Send`
    - `Sync`：若类型可通过 `&T` 在线程间安全**共享**，则该类型是 `Sync`
- 大多数类型自动是 `Send + Sync`。值得注意的例外：
    - `Rc<T>` **既非** Send **也非** Sync（在线程中使用 `Arc<T>`）
    - `Cell<T>` 和 `RefCell<T>` **不是** Sync（使用 `Mutex<T>` 或 `RwLock<T>`）
    - 裸指针（`*const T`、`*mut T`）**既非** Send **也非** Sync
- 这就是为什么编译器阻止你跨线程使用 `Rc<T>`——它根本没有实现 `Send`
- `Arc<Mutex<T>>` 是 `Rc<RefCell<T>>` 的线程安全等价物

> **直觉** *（Jon Gjengset）*：把值想象成玩具。
> **`Send`** = 你可以把玩具**送给**另一个孩子（线程）——转移所有权是安全的。
> **`Sync`** = 你可以**让别人同时玩**你的玩具——共享引用是安全的。
> `Rc<T>` 有脆弱（非原子）引用计数；转移或共享它会破坏计数，因此它既非 `Send` 也非 `Sync`。


# 练习：多线程词频统计 {#exercise-multi-threaded-word-count}

🔴 **挑战**——结合线程、Arc、Mutex 和 HashMap

- 给定 `Vec<String>` 文本行，为每行 spawn 一个线程统计该行词数
- 使用 `Arc<Mutex<HashMap<String, usize>>>` 收集结果
- 打印所有行的总词数
- **Bonus**：尝试用通道（`mpsc`）而非共享状态实现

<details><summary>Solution (click to expand)</summary>

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let lines = vec![
        "the quick brown fox".to_string(),
        "jumps over the lazy dog".to_string(),
        "the fox is quick".to_string(),
    ];

    let word_counts: Arc<Mutex<HashMap<String, usize>>> =
        Arc::new(Mutex::new(HashMap::new()));

    let mut handles = vec![];
    for line in &lines {
        let line = line.clone();
        let counts = Arc::clone(&word_counts);
        handles.push(thread::spawn(move || {
            for word in line.split_whitespace() {
                let mut map = counts.lock().unwrap();
                *map.entry(word.to_lowercase()).or_insert(0) += 1;
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let counts = word_counts.lock().unwrap();
    let total: usize = counts.values().sum();
    println!("Word frequencies: {counts:#?}");
    println!("Total words: {total}");
}
// Output (order may vary):
// Word frequencies: {
//     "the": 3,
//     "quick": 2,
//     "brown": 1,
//     "fox": 2,
//     "jumps": 1,
//     "over": 1,
//     "lazy": 1,
//     "dog": 1,
//     "is": 1,
// }
// Total words: 13
```

</details>

