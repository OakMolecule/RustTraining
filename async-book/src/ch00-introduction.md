# Async Rust：从 Future 到生产实践

> **版权与署名说明（非官方译本）**
> - 本文档为 [microsoft/RustTraining](https://github.com/microsoft/RustTraining) 中 `async-book` 的中文翻译与改编版本。
> - 译文主要由 AI 生成，未经过人工校对与修订。
> - 原作版权归 Microsoft Corporation 所有；文档许可为 [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)。
> - 本译本为社区非官方版本，不代表 Microsoft 官方立场。

## 讲师介绍

- Microsoft SCHIE（Silicon and Cloud Hardware Infrastructure Engineering，硅与云硬件基础设施工程）团队 Principal Firmware Architect
- 资深行业专家，专长于安全、系统编程（固件、操作系统、虚拟机监控程序）、CPU 与平台架构，以及 C++ 系统开发
- 2017 年起在 Rust 上编程（@AWS EC2），此后一直热爱这门语言

---

Rust 异步编程深度指南。与大多数从 `tokio::main` 入手、对内部机制一笔带过的 async 教程不同，本指南从第一性原理出发——`Future` Trait、轮询（polling）、状态机——逐步推进到真实世界的模式、运行时选型与生产陷阱。

## 适合谁读

- 能写同步 Rust，但觉得 async 令人困惑的 Rust 开发者
- 来自 C#、Go、Python 或 JavaScript、熟悉 `async/await` 但不了解 Rust 模型的开发者
- 曾被 `Future is not Send`、`Pin<Box<dyn Future>>`，或「程序为什么挂住了？」困扰的任何人

## 前置知识

你应熟悉：

- 所有权（Ownership）、借用（Borrowing）与生命周期（Lifetime）
- Trait 与泛型（包括 `impl Trait`）
- 使用 `Result<T, E>` 与 `?` 运算符
- 基本多线程（`std::thread::spawn`、`Arc`、`Mutex`）

无需先前的 async Rust 经验。

## 如何使用本书

**首次阅读请按顺序通读。** 第一至三部分相互依赖。每章标注：

| 符号 | 含义 |
|--------|---------|
| 🟢 | 入门 — 基础概念 |
| 🟡 | 中级 — 需先读前面章节 |
| 🔴 | 高级 — 深入内部机制或生产模式 |

每章包含：

- 顶部的 **「你将学到」** 块
- 面向视觉学习者的 **Mermaid 图**
- 带隐藏解答的 **内联练习**
- 总结核心思想的 **要点回顾**
- 指向相关章节的 **交叉引用**

## 进度建议

| 章节 | 主题 | 建议用时 | 检查点 |
|----------|-------|----------------|------------|
| 1–5 | 异步如何工作 | 6–8 小时 | 能解释 `Future`、`Poll`、`Pin`，以及 Rust 为何没有内置运行时 |
| 6–10 | 生态系统 | 6–8 小时 | 能手动构建 future、选择运行时、使用 tokio API |
| 11–13 | 生产级异步 | 6–8 小时 | 能编写生产级 async 代码，包括 stream、正确错误处理与优雅关闭 |
| 毕业项目 | 聊天服务器 | 4–6 小时 | 已构建整合所有概念的真实 async 应用 |

**总预估时间：22–30 小时**

## 如何完成练习

每个内容章节都有内联练习。毕业项目（第 17 章）将所有内容整合为单一项目。为获得最大学习效果：

1. **展开解答前先尝试练习** — 挣扎才是学习发生的地方
2. **亲手敲代码，不要复制粘贴** — 肌肉记忆对 Rust 语法很重要
3. **运行每个示例** — 使用 `cargo new async-exercises`，边学边测

## 目录

### 第一部分：异步如何工作

- [1. Rust 中异步为何不同](ch01-why-async-is-different-in-rust.md) 🟢 — 根本差异：Rust 没有内置运行时
- [2. Future Trait](ch02-the-future-trait.md) 🟡 — `poll()`、`Waker`，以及使一切运转的契约
- [3. Poll 如何工作](ch03-how-poll-works.md) 🟡 — 轮询状态机与最小执行器
- [4. Pin 与 Unpin](ch04-pin-and-unpin.md) 🔴 — 自引用结构体为何需要固定（pinning）
- [5. 状态机揭秘](ch05-the-state-machine-reveal.md) 🟢 — 编译器从 `async fn` 实际生成了什么

### 第二部分：生态系统

- [6. 手动构建 Future](ch06-building-futures-by-hand.md) 🟡 — 从零实现 TimerFuture、Join、Select
- [7. 执行器与运行时](ch07-executors-and-runtimes.md) 🟡 — tokio、smol、async-std、embassy — 如何选择
- [8. Tokio 深入解析](ch08-tokio-deep-dive.md) 🟡 — 运行时风格、spawn、channel、同步原语
- [9. 何时不该用 Tokio](ch09-when-tokio-isnt-the-right-fit.md) 🟡 — LocalSet、FuturesUnordered、与运行时无关的设计
- [10. 异步 Trait](ch10-async-traits.md) 🟡 — RPITIT、dyn 分发、`trait_variant`、异步闭包

### 第三部分：生产级异步

- [11. Stream 与 AsyncIterator](ch11-streams-and-asynciterator.md) 🟡 — 异步迭代、AsyncRead/Write、stream 组合子
- [12. 常见陷阱](ch12-common-pitfalls.md) 🔴 — 9 个生产 bug 及如何避免
- [13. 生产模式](ch13-production-patterns.md) 🔴 — 优雅关闭、背压、Tower 中间件
- [14. 异步是优化，而非架构](ch14-async-is-an-optimization-not-an-architecture.md) 🔴 — 同步核心 / 异步外壳、函数着色代价

### 附录

- [总结与参考卡](ch16-summary-and-reference-card.md) — 快速查阅表与决策树
- [毕业项目：异步聊天服务器](ch17-capstone-project.md) — 构建完整 async 应用

***

