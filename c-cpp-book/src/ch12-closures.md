## Rust 闭包 {#rust-closures}

> **你将学到：** 闭包作为匿名函数、三种捕获 Trait（`Fn`、`FnMut`、`FnOnce`）、`move` 闭包，以及 Rust 闭包与 C++ lambda 的对比——自动捕获分析，而非手动的 `[&]`/`[=]` 指定。

- 闭包是可以捕获其环境的匿名函数
    - C++ 等价物：lambda（`[&](int x) { return x + 1; }`）
    - 关键差异：Rust 闭包有**三种**捕获 Trait（`Fn`、`FnMut`、`FnOnce`），由编译器自动选择
    - C++ 捕获模式（`[=]`、`[&]`、`[this]`）是手动的且容易出错（悬垂的 `[&]`！）
    - Rust 的借用检查器在编译期防止悬垂捕获
- 闭包可通过 `||` 符号识别。类型参数包含在 `||` 内，可使用类型推断
- 闭包常与迭代器配合使用（下一主题）
```rust
fn add_one(x: u32) -> u32 {
    x + 1
}
fn main() {
    let add_one_v1 = |x : u32| {x + 1}; // Explicitly specified type
    let add_one_v2 = |x| {x + 1};   // Type is inferred from call site
    let add_one_v3 = |x| x+1;   // Permitted for single line functions
    println!("{} {} {} {}", add_one(42), add_one_v1(42), add_one_v2(42), add_one_v3(42) );
}
```


# 练习：闭包与捕获 {#exercise-closures-and-capturing}

🟡 **进阶**

- 创建一个从外围作用域捕获 `String` 并追加内容的闭包（提示：使用 `move`）
- 创建一个闭包向量：`Vec<Box<dyn Fn(i32) -> i32>>`，包含加 1、乘 2 和平方输入的闭包。遍历该向量，对每个闭包应用数字 5

<details><summary>Solution (click to expand)</summary>

```rust
fn main() {
    // Part 1: Closure that captures and appends to a String
    let mut greeting = String::from("Hello");
    let mut append = |suffix: &str| {
        greeting.push_str(suffix);
    };
    append(", world");
    append("!");
    println!("{greeting}");  // "Hello, world!"

    // Part 2: Vector of closures
    let operations: Vec<Box<dyn Fn(i32) -> i32>> = vec![
        Box::new(|x| x + 1),      // add 1
        Box::new(|x| x * 2),      // multiply by 2
        Box::new(|x| x * x),      // square
    ];

    let input = 5;
    for (i, op) in operations.iter().enumerate() {
        println!("Operation {i} on {input}: {}", op(input));
    }
}
// Output:
// Hello, world!
// Operation 0 on 5: 6
// Operation 1 on 5: 10
// Operation 2 on 5: 25
```

</details>

# Rust 迭代器 {#rust-iterators}
- 迭代器是 Rust 最强大的特性之一。它们为对集合执行操作提供了非常优雅的方法，包括过滤（```filter()```）、转换（```map()```）、过滤并映射（```filter_map()```）、搜索（```find()```）等等
- 在以下示例中，```|&x| *x >= 42``` 是执行相同比较的闭包。```|x| println!("{x}")``` 是另一个闭包
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    for x in &a {
        if *x >= 42 {
            println!("{x}");
        }
    }
    // Same as above
    a.iter().filter(|&x| *x >= 42).for_each(|x| println!("{x}"))
}
```

# Rust 迭代器
- 迭代器的一个关键特性是大多数是 ```lazy``` 的，即在被求值之前不会做任何事。例如，```a.iter().filter(|&x| *x >= 42);``` 没有 ```for_each``` 就*不会*做任何事。Rust 编译器在检测到这种情况时会发出显式警告
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    // Add one to each element and print it
    let _ = a.iter().map(|x|x + 1).for_each(|x|println!("{x}"));
    let found = a.iter().find(|&x|*x == 42);
    println!("{found:?}");
    // Count elements
    let count = a.iter().count();
    println!("{count}");
}
```

# Rust 迭代器
- ```collect()``` 方法可用于将结果收集到单独的集合中
    - 在下方，```Vec<_>``` 中的 ```_``` 相当于 ```map``` 返回类型的通配符。例如，我们甚至可以从 ```map``` 返回 ```String```
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    let squared_a : Vec<_> = a.iter().map(|x|x*x).collect();
    for x in &squared_a {
        println!("{x}");
    }
    let squared_a_strings : Vec<_> = a.iter().map(|x|(x*x).to_string()).collect();
    // These are actually string representations
    for x in &squared_a_strings {
        println!("{x}");
    }
}
```

# 练习：Rust 迭代器 {#exercise-rust-iterators}

🟢 **入门**
- 创建一个由奇数和偶数元素组成的整数数组。遍历数组，将其拆分为两个向量，分别包含偶数和奇数元素
- 能否一次遍历完成（提示：使用 ```partition()```）？

<details><summary>Solution (click to expand)</summary>

```rust
fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Approach 1: Manual iteration
    let mut evens = Vec::new();
    let mut odds = Vec::new();
    for n in numbers {
        if n % 2 == 0 {
            evens.push(n);
        } else {
            odds.push(n);
        }
    }
    println!("Evens: {evens:?}");
    println!("Odds:  {odds:?}");

    // Approach 2: Single pass with partition()
    let (evens, odds): (Vec<i32>, Vec<i32>) = numbers
        .into_iter()
        .partition(|n| *n % 2 == 0);
    println!("Evens (partition): {evens:?}");
    println!("Odds  (partition): {odds:?}");
}
// Output:
// Evens: [2, 4, 6, 8, 10]
// Odds:  [1, 3, 5, 7, 9]
// Evens (partition): [2, 4, 6, 8, 10]
// Odds  (partition): [1, 3, 5, 7, 9]
```

</details>

> **生产模式**：参见[用闭包折叠赋值金字塔](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids-with-closures)，了解生产 Rust 代码中的真实迭代器链（`.map().collect()`、`.filter().collect()`、`.find_map()`）。

### 迭代器强力工具：替代 C++ 循环的方法

以下迭代器适配器在*生产* Rust 代码中*广泛使用*。C++ 有
`<algorithm>` 和 C++20 ranges，但 Rust 的迭代器链更可组合、
也更常用。

#### `enumerate`——索引 + 值（替代 `for (int i = 0; ...)`）

```rust
let sensors = vec!["temp0", "temp1", "temp2"];
for (idx, name) in sensors.iter().enumerate() {
    println!("Sensor {idx}: {name}");
}
// Sensor 0: temp0
// Sensor 1: temp1
// Sensor 2: temp2
```

C++ 等价物：`for (size_t i = 0; i < sensors.size(); ++i) { auto& name = sensors[i]; ... }`

#### `zip`——配对两个迭代器的元素（替代并行索引循环）

```rust
let names = ["gpu0", "gpu1", "gpu2"];
let temps = [72.5, 68.0, 75.3];

let report: Vec<String> = names.iter()
    .zip(temps.iter())
    .map(|(name, temp)| format!("{name}: {temp}°C"))
    .collect();
println!("{report:?}");
// ["gpu0: 72.5°C", "gpu1: 68.0°C", "gpu2: 75.3°C"]

// Stops at the shorter iterator — no out-of-bounds risk
```

C++ 等价物：`for (size_t i = 0; i < std::min(names.size(), temps.size()); ++i) { ... }`

#### `flat_map`——map + 展平嵌套集合

```rust
// Each GPU has multiple PCIe BDFs; collect all BDFs across all GPUs
let gpu_bdfs = vec![
    vec!["0000:01:00.0", "0000:02:00.0"],
    vec!["0000:41:00.0"],
    vec!["0000:81:00.0", "0000:82:00.0"],
];

let all_bdfs: Vec<&str> = gpu_bdfs.iter()
    .flat_map(|bdfs| bdfs.iter().copied())
    .collect();
println!("{all_bdfs:?}");
// ["0000:01:00.0", "0000:02:00.0", "0000:41:00.0", "0000:81:00.0", "0000:82:00.0"]
```

C++ 等价物：嵌套 `for` 循环推入单个向量。

#### `chain`——连接两个迭代器

```rust
let critical_gpus = vec!["gpu0", "gpu3"];
let warning_gpus = vec!["gpu1", "gpu5"];

// Process all flagged GPUs, critical first
for gpu in critical_gpus.iter().chain(warning_gpus.iter()) {
    println!("Flagged: {gpu}");
}
```

#### `windows` 和 `chunks`——切片上的滑动/固定大小视图

```rust
let temps = [70, 72, 75, 73, 71, 68, 65];

// windows(3): sliding window of size 3 — detect trends
let rising = temps.windows(3)
    .any(|w| w[0] < w[1] && w[1] < w[2]);
println!("Rising trend detected: {rising}"); // true (70 < 72 < 75)

// chunks(2): fixed-size groups — process in pairs
for pair in temps.chunks(2) {
    println!("Pair: {pair:?}");
}
// Pair: [70, 72]
// Pair: [75, 73]
// Pair: [71, 68]
// Pair: [65]       ← last chunk can be smaller
```

C++ 等价物：手动索引运算 `i` 和 `i+1`/`i+2`。

#### `fold`——累加为单个值（替代 `std::accumulate`）

```rust
let errors = vec![
    ("gpu0", 3u32),
    ("gpu1", 0),
    ("gpu2", 7),
    ("gpu3", 1),
];

// Count total errors and build summary in one pass
let (total, summary) = errors.iter().fold(
    (0u32, String::new()),
    |(count, mut s), (name, errs)| {
        if *errs > 0 {
            s.push_str(&format!("{name}:{errs} "));
        }
        (count + errs, s)
    },
);
println!("Total errors: {total}, details: {summary}");
// Total errors: 11, details: gpu0:3 gpu2:7 gpu3:1
```

#### `scan`——有状态的转换（运行总和、增量检测）

```rust
let readings = [100, 105, 103, 110, 108];

// Compute deltas between consecutive readings
let deltas: Vec<i32> = readings.iter()
    .scan(None::<i32>, |prev, &val| {
        let delta = prev.map(|p| val - p);
        *prev = Some(val);
        Some(delta)
    })
    .flatten()  // Remove the initial None
    .collect();
println!("Deltas: {deltas:?}"); // [5, -2, 7, -2]
```

#### 快速参考：C++ 循环 → Rust 迭代器

| **C++ 模式** | **Rust 迭代器** | **示例** |
|----------------|------------------|------------|
| `for (int i = 0; i < v.size(); i++)` | `.enumerate()` | `v.iter().enumerate()` |
| 带索引的并行迭代 | `.zip()` | `a.iter().zip(b.iter())` |
| 嵌套循环 → 扁平结果 | `.flat_map()` | `vecs.iter().flat_map(\|v\| v.iter())` |
| 连接两个容器 | `.chain()` | `a.iter().chain(b.iter())` |
| 滑动窗口 `v[i..i+n]` | `.windows(n)` | `v.windows(3)` |
| 按固定大小分组处理 | `.chunks(n)` | `v.chunks(4)` |
| `std::accumulate` / 手动累加器 | `.fold()` | `.fold(init, \|acc, x\| ...)` |
| 运行总和 / 增量跟踪 | `.scan()` | `.scan(state, \|s, x\| ...)` |
| `while (it != end && count < n) { ++it; ++count; }` | `.take(n)` | `.iter().take(5)` |
| `while (it != end && !pred(*it)) { ++it; }` | `.skip_while()` | `.skip_while(\|x\| x < &threshold)` |
| `std::any_of` | `.any()` | `.iter().any(\|x\| x > &limit)` |
| `std::all_of` | `.all()` | `.iter().all(\|x\| x.is_valid())` |
| `std::none_of` | `!.any()` | `!iter.any(\|x\| x.failed())` |
| `std::count_if` | `.filter().count()` | `.filter(\|x\| x > &0).count()` |
| `std::min_element` / `std::max_element` | `.min()` / `.max()` | `.iter().max()` → `Option<&T>` |
| `std::unique` | `.dedup()`（对已排序） | `v.dedup()`（对 Vec 原地操作） |

### 练习：迭代器链

给定传感器数据 `Vec<(String, f64)>`（名称、温度），编写**一条**
迭代器链：
1. 过滤温度 > 80.0 的传感器
2. 按温度降序排序
3. 将每个格式化为 `"{name}: {temp}°C [ALARM]"`
4. 收集为 `Vec<String>`

提示：排序需要 `Vec`，因此 `.sort_by()` 之前需要 `.collect()`。

<details><summary>Solution (click to expand)</summary>

```rust
fn alarm_report(sensors: &[(String, f64)]) -> Vec<String> {
    let mut hot: Vec<_> = sensors.iter()
        .filter(|(_, temp)| *temp > 80.0)
        .collect();
    hot.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    hot.iter()
        .map(|(name, temp)| format!("{name}: {temp}°C [ALARM]"))
        .collect()
}

fn main() {
    let sensors = vec![
        ("gpu0".to_string(), 72.5),
        ("gpu1".to_string(), 85.3),
        ("gpu2".to_string(), 91.0),
        ("gpu3".to_string(), 78.0),
        ("gpu4".to_string(), 88.7),
    ];
    for line in alarm_report(&sensors) {
        println!("{line}");
    }
}
// Output:
// gpu2: 91°C [ALARM]
// gpu4: 88.7°C [ALARM]
// gpu1: 85.3°C [ALARM]
```

</details>

----

# Rust 迭代器
- ```Iterator``` Trait 用于为用户定义类型实现迭代（https://doc.rust-lang.org/std/iter/trait.IntoIterator.html）
    - 在示例中，我们将为斐波那契数列实现迭代器，它以 1, 1, 2, ... 开始，后继数是前两个数之和
    - ```Iterator``` 中的 ```associated type```（```type Item = u32;```）定义迭代器的输出类型（```u32```）
    - ```next()``` 方法包含实现迭代器的逻辑。此例中，所有状态信息都在 ```Fibonacci``` 结构体中
    - 我们也可以实现名为 ```IntoIterator``` 的另一个 Trait，为更专门的迭代器实现 ```into_iter()``` 方法
    - [▶ 在 Rust Playground 中尝试](https://play.rust-lang.org/)

