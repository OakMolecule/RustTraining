# 面向 C/C++ 程序员的 Rust 入门课程

> **版权与署名说明（非官方译本）**
> - 本文档为 [microsoft/RustTraining](https://github.com/microsoft/RustTraining) 中 `c-cpp-book` 的中文翻译与改编版本。
> - 译文主要由 AI 生成，未经过人工校对与修订。
> - 原作版权归 Microsoft Corporation 所有；文档许可为 [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)。
> - 本译本为社区非官方版本，不代表 Microsoft 官方立场。

## 课程概览
- 课程概览
    - 选择 Rust 的理由（从 C 和 C++ 两个视角）
    - 本地安装
    - 类型、函数、控制流、模式匹配
    - 模块、Cargo
    - Trait、泛型
    - 集合、错误处理
    - 闭包、内存管理、生命周期、智能指针
    - 并发
    - Unsafe Rust，包括外部函数接口（Foreign Function Interface，FFI）
    - `no_std` 与面向固件团队的嵌入式 Rust 要点
    - 案例研究：真实世界的 C++ 到 Rust 翻译模式
- 本课程不涵盖 `async` Rust——请参阅配套 [Async Rust Training](../async-book/)，了解 future、执行器、`Pin`、tokio 及生产级异步模式

---

# 自学指南

本材料既可作为讲师授课课程，也适合自学。如果你独自学习，以下建议可帮助你充分利用：

**进度建议：**

| 章节 | 主题 | 建议用时 | 检查点 |
|----------|-------|---------------|------------|
| 1–4 | 环境搭建、类型、控制流 | 1 天 | 能编写 CLI 温度转换器 |
| 5–7 | 数据结构、所有权 | 1–2 天 | 能解释 *为什么* `let s2 = s1` 会使 `s1` 失效 |
| 8–9 | 模块、错误处理 | 1 天 | 能创建多文件项目并用 `?` 传播错误 |
| 10–12 | Trait、泛型、闭包 | 1–2 天 | 能编写带 Trait 约束的泛型函数 |
| 13–14 | 并发、unsafe/FFI | 1 天 | 能用 `Arc<Mutex<T>>` 编写线程安全计数器 |
| 15–16 | 深入专题 | 自定进度 | 参考资料——在需要时阅读 |
| 17–19 | 最佳实践与参考 | 自定进度 | 编写真实代码时查阅 |

**如何使用练习：**
- 每章都有动手练习，难度标注为：🟢 入门、🟡 中级、🔴 挑战
- **务必在展开答案前先尝试练习。** 与借用检查器较劲是学习的一部分——编译器的错误信息就是你的老师
- 如果卡住超过 15 分钟，展开答案、研读后关闭，再从头试一次
- [Rust Playground](https://play.rust-lang.org/) 无需本地安装即可运行代码

**遇到瓶颈时：**
- 仔细阅读编译器错误信息——Rust 的错误提示非常出色
- 重读相关章节；所有权（第 7 章）等概念往往在第二遍才豁然开朗
- [Rust 标准库文档](https://doc.rust-lang.org/std/) 质量很高——可搜索任意类型或方法
- 异步模式请参阅配套 [Async Rust Training](../async-book/)

---

# 目录

## 第一部分 — 基础

### 1. 引言与动机
- [讲师介绍与总体方法](ch01-introduction-and-motivation.md#speaker-intro-and-general-approach)
- [选择 Rust 的理由](ch01-introduction-and-motivation.md#the-case-for-rust)
- [Rust 如何解决这些问题？](ch01-introduction-and-motivation.md#how-does-rust-address-these-issues)
- [Rust 的其他独特卖点与特性](ch01-introduction-and-motivation.md#other-rust-usps-and-features)
- [快速参考：Rust vs C/C++](ch01-introduction-and-motivation.md#quick-reference-rust-vs-cc)
- [为什么 C/C++ 开发者需要 Rust](ch01-1-why-c-cpp-developers-need-rust.md)
  - [Rust 消除了什么——完整清单](ch01-1-why-c-cpp-developers-need-rust.md#what-rust-eliminates--the-complete-list)
  - [C 与 C++ 共同面临的问题](ch01-1-why-c-cpp-developers-need-rust.md#the-problems-shared-by-c-and-c)
  - [C++ 在此基础上又增加了更多问题](ch01-1-why-c-cpp-developers-need-rust.md#c-adds-more-problems-on-top)
  - [Rust 如何解决这一切](ch01-1-why-c-cpp-developers-need-rust.md#how-rust-addresses-all-of-this)

### 2. 快速上手
- [少说多做：给我看代码](ch02-getting-started.md#enough-talk-already-show-me-some-code)
- [Rust 本地安装](ch02-getting-started.md#rust-local-installation)
- [Rust 包（crate）](ch02-getting-started.md#rust-packages-crates)
- [示例：cargo 与 crate](ch02-getting-started.md#example-cargo-and-crates)

### 3. 基本类型与变量
- [Rust 内置类型](ch03-built-in-types.md#built-in-rust-types)
- [Rust 类型指定与赋值](ch03-built-in-types.md#rust-type-specification-and-assignment)
- [Rust 类型指定与推断](ch03-built-in-types.md#rust-type-specification-and-inference)
- [Rust 变量与可变性](ch03-built-in-types.md#rust-variables-and-mutability)

### 4. 控制流
- [Rust `if` 关键字](ch04-control-flow.md#rust-if-keyword)
- [Rust 使用 `while` 和 `for` 的循环](ch04-control-flow.md#rust-loops-using-while-and-for)
- [Rust 使用 `loop` 的循环](ch04-control-flow.md#rust-loops-using-loop)
- [Rust 表达式块](ch04-control-flow.md#rust-expression-blocks)

### 5. 数据结构与集合
- [Rust 数组类型](ch05-data-structures.md#rust-array-type)
- [Rust 元组](ch05-data-structures.md#rust-tuples)
- [Rust 引用](ch05-data-structures.md#rust-references)
- [C++ 引用 vs Rust 引用——关键差异](ch05-data-structures.md#c-references-vs-rust-references--key-differences)
- [Rust 切片](ch05-data-structures.md#rust-slices)
- [Rust 常量与静态变量](ch05-data-structures.md#rust-constants-and-statics)
- [Rust 字符串：`String` vs `&str`](ch05-data-structures.md#rust-strings-string-vs-str)
- [Rust 结构体](ch05-data-structures.md#rust-structs)
- [Rust `Vec<T>`](ch05-data-structures.md#rust-vec-type)
- [Rust `HashMap`](ch05-data-structures.md#rust-hashmap-type)
- [练习：Vec 与 HashMap](ch05-data-structures.md#exercise-vec-and-hashmap)

### 6. 模式匹配与枚举
- [Rust 枚举类型](ch06-enums-and-pattern-matching.md#rust-enum-types)
- [Rust `match` 语句](ch06-enums-and-pattern-matching.md#rust-match-statement)
- [练习：使用 match 和 enum 实现加减](ch06-enums-and-pattern-matching.md#exercise-implement-add-and-subtract-using-match-and-enum)

### 7. 所有权与内存管理
- [Rust 内存管理](ch07-ownership-and-borrowing.md#rust-memory-management)
- [Rust 所有权、借用与生命周期](ch07-ownership-and-borrowing.md#rust-ownership-borrowing-and-lifetimes)
- [Rust 移动语义](ch07-ownership-and-borrowing.md#rust-move-semantics)
- [Rust `Clone`](ch07-ownership-and-borrowing.md#rust-clone)
- [Rust `Copy` Trait](ch07-ownership-and-borrowing.md#rust-copy-trait)
- [Rust `Drop` Trait](ch07-ownership-and-borrowing.md#rust-drop-trait)
- [练习：Move、Copy 与 Drop](ch07-ownership-and-borrowing.md#exercise-move-copy-and-drop)
- [Rust 生命周期与借用](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-and-borrowing)
- [Rust 生命周期标注](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-annotations)
- [练习：带生命周期的切片存储](ch07-1-lifetimes-and-borrowing-deep-dive.md#exercise-slice-storage-with-lifetimes)
- [生命周期省略规则深入](ch07-1-lifetimes-and-borrowing-deep-dive.md#lifetime-elision-rules-deep-dive)
- [Rust `Box<T>`](ch07-2-smart-pointers-and-interior-mutability.md#rust-boxt)
- [内部可变性：`Cell<T>` 与 `RefCell<T>`](ch07-2-smart-pointers-and-interior-mutability.md#interior-mutability-cellt-and-refcellt)
- [共享所有权：`Rc<T>`](ch07-2-smart-pointers-and-interior-mutability.md#shared-ownership-rct)
- [练习：共享所有权与内部可变性](ch07-2-smart-pointers-and-interior-mutability.md#exercise-shared-ownership-and-interior-mutability)

### 8. 模块与 Crate
- [Rust crate 与模块](ch08-crates-and-modules.md#rust-crates-and-modules)
- [练习：模块与函数](ch08-crates-and-modules.md#exercise-modules-and-functions)
- [工作区与 crate（包）](ch08-crates-and-modules.md#workspaces-and-crates-packages)
- [练习：使用工作区与包依赖](ch08-crates-and-modules.md#exercise-using-workspaces-and-package-dependencies)
- [使用 crates.io 社区 crate](ch08-crates-and-modules.md#using-community-crates-from-cratesio)
- [Crate 依赖与 SemVer](ch08-crates-and-modules.md#crates-dependencies-and-semver)
- [练习：使用 rand crate](ch08-crates-and-modules.md#exercise-using-the-rand-crate)
- [Cargo.toml 与 Cargo.lock](ch08-crates-and-modules.md#cargotoml-and-cargolock)
- [Cargo 测试功能](ch08-crates-and-modules.md#cargo-test-feature)
- [Cargo 其他功能](ch08-crates-and-modules.md#other-cargo-features)
- [测试模式](ch08-1-testing-patterns.md)

### 9. 错误处理
- [将枚举与 Option 和 Result 联系起来](ch09-error-handling.md#connecting-enums-to-option-and-result)
- [Rust `Option` 类型](ch09-error-handling.md#rust-option-type)
- [Rust `Result` 类型](ch09-error-handling.md#rust-result-type)
- [练习：使用 Option 实现 log() 函数](ch09-error-handling.md#exercise-log-function-implementation-with-option)
- [Rust 错误处理](ch09-error-handling.md#rust-error-handling)
- [练习：错误处理](ch09-error-handling.md#exercise-error-handling)
- [错误处理最佳实践](ch09-1-error-handling-best-practices.md)

### 10. Trait 与泛型
- [Rust Trait](ch10-traits.md#rust-traits)
- [C++ 运算符重载 → Rust `std::ops` Trait](ch10-traits.md#c-operator-overloading--rust-stdops-traits)
- [练习：Logger Trait 实现](ch10-traits.md#exercise-logger-trait-implementation)
- [何时使用 enum vs `dyn Trait`](ch10-traits.md#when-to-use-enum-vs-dyn-trait)
- [练习：翻译前先思考](ch10-traits.md#exercise-think-before-you-translate)
- [Rust 泛型](ch10-1-generics.md#rust-generics)
- [练习：泛型](ch10-1-generics.md#exercise-generics)
- [结合 Rust Trait 与泛型](ch10-1-generics.md#combining-rust-traits-and-generics)
- [数据类型中的 Rust Trait 约束](ch10-1-generics.md#rust-traits-constraints-in-data-types)
- [练习：Trait 约束与泛型](ch10-1-generics.md#exercise-traits-constraints-and-generics)
- [Rust 类型状态模式与泛型](ch10-1-generics.md#rust-type-state-pattern-and-generics)
- [Rust 建造者模式](ch10-1-generics.md#rust-builder-pattern)

### 11. 类型系统高级特性
- [Rust `From` 与 `Into` Trait](ch11-from-and-into-traits.md#rust-from-and-into-traits)
- [练习：From 与 Into](ch11-from-and-into-traits.md#exercise-from-and-into)
- [Rust `Default` Trait](ch11-from-and-into-traits.md#rust-default-trait)
- [其他 Rust 类型转换](ch11-from-and-into-traits.md#other-rust-type-conversions)

### 12. 函数式编程
- [Rust 闭包](ch12-closures.md#rust-closures)
- [练习：闭包与捕获](ch12-closures.md#exercise-closures-and-capturing)
- [Rust 迭代器](ch12-closures.md#rust-iterators)
- [练习：Rust 迭代器](ch12-closures.md#exercise-rust-iterators)
- [迭代器强力工具参考](ch12-1-iterator-power-tools.md#iterator-power-tools-reference)

### 13. 并发
- [Rust 并发](ch13-concurrency.md#rust-concurrency)
- [Rust 如何防止数据竞争：Send 与 Sync](ch13-concurrency.md#why-rust-prevents-data-races-send-and-sync)
- [练习：多线程词频统计](ch13-concurrency.md#exercise-multi-threaded-word-count)

### 14. Unsafe Rust 与 FFI
- [Unsafe Rust](ch14-unsafe-rust-and-ffi.md#unsafe-rust)
- [简单 FFI 示例](ch14-unsafe-rust-and-ffi.md#simple-ffi-example-rust-library-function-consumed-by-c)
- [复杂 FFI 示例](ch14-unsafe-rust-and-ffi.md#complex-ffi-example)
- [确保 unsafe 代码的正确性](ch14-unsafe-rust-and-ffi.md#ensuring-correctness-of-unsafe-code)
- [练习：编写安全的 FFI 包装](ch14-unsafe-rust-and-ffi.md#exercise-writing-a-safe-ffi-wrapper)

## 第二部分 — 深入专题

### 15. `no_std` — 裸机 Rust
- [什么是 `no_std`？](ch15-no_std-rust-without-the-standard-library.md#what-is-no_std)
- [何时使用 `no_std` vs `std`](ch15-no_std-rust-without-the-standard-library.md#when-to-use-no_std-vs-std)
- [练习：`no_std` 环形缓冲区](ch15-no_std-rust-without-the-standard-library.md#exercise-no_std-ring-buffer)
- [嵌入式深入](ch15-1-embedded-deep-dive.md)

### 16. 案例研究：真实世界的 C++ 到 Rust 翻译
- [案例研究 1：继承层次 → 枚举分发](ch16-case-studies.md#case-study-1-inheritance-hierarchy--enum-dispatch)
- [案例研究 2：`shared_ptr` 树 → Arena/索引模式](ch16-case-studies.md#case-study-2-shared_ptr-tree--arenaindex-pattern)
- [案例研究 3：框架通信 → 生命周期借用](ch16-cases-3-5-lifetime-borrowing.md#case-study-3-framework-communication--lifetime-borrowing)
- [案例研究 4：上帝对象 → 可组合状态](ch16-cases-3-5-lifetime-borrowing.md#case-study-4-god-object--composable-state)
- [案例研究 5：Trait 对象——何时才合适](ch16-cases-3-5-lifetime-borrowing.md#case-study-5-trait-objects--when-they-are-right)

## 第三部分 — 最佳实践与参考

### 17. 最佳实践
- [Rust 最佳实践摘要](ch17-best-practices.md#rust-best-practices-summary)
- [避免过度 `clone()`](ch17-1-avoiding-excessive-clone.md#avoiding-excessive-clone)
- [避免未检查的索引](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing)
- [折叠赋值金字塔](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids)
- [综合练习：诊断事件管道](ch17-3-collapsing-assignment-pyramids.md#capstone-exercise-diagnostic-event-pipeline)
- [日志与追踪生态](ch17-4-logging-and-tracing-ecosystem.md#logging-and-tracing-ecosystem)

### 18. C++ → Rust 语义深入
- [类型转换、预处理器、模块、`volatile`、`static`、`constexpr`、SFINAE 等](ch18-cpp-rust-semantic-deep-dives.md)

### 19. Rust 宏
- [声明式宏（`macro_rules!`）](ch19-macros.md#declarative-macros-with-macro_rules)
- [常用标准库宏](ch19-macros.md#common-standard-library-macros)
- [派生宏](ch19-macros.md#derive-macros)
- [属性宏](ch19-macros.md#attribute-macros)
- [过程宏](ch19-macros.md#procedural-macros-conceptual-overview)
- [如何选择：宏 vs 函数 vs 泛型](ch19-macros.md#when-to-use-what-macros-vs-functions-vs-generics)
- [练习](ch19-macros.md#exercises)
