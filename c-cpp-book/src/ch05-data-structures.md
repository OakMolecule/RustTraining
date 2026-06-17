### Rust 数组类型 {#rust-array-type}

> **你将学到：** Rust 的核心数据结构——数组、元组、切片、字符串、结构体、`Vec` 和 `HashMap`。本章内容较密集；重点理解 `String` 与 `&str` 的区别，以及结构体的工作方式。引用与借用将在第 7 章深入讲解。

- 数组包含固定数量的同类型元素
    - 与 Rust 中其他类型一样，数组默认不可变（除非使用 `mut`）
    - 数组使用 `[]` 索引，并会进行边界检查。可用 `len()` 方法获取数组长度
```rust
    fn get_index(y : usize) -> usize {
        y+1        
    }
    
    fn main() {
        // Initializes an array of 3 elements and sets all to 42
        let a : [u8; 3] = [42; 3];
        // Alternative syntax
        // let a = [42u8, 42u8, 42u8];
        for x in a {
            println!("{x}");
        }
        let y = get_index(a.len());
        // Commenting out the below will cause a panic
        //println!("{}", a[y]);
    }
```

----
### Rust 数组类型（续）
- 数组可以嵌套
    - Rust 内置多种打印格式化器。下面示例中，```:?``` 是 ```debug``` 打印格式化器。```:#?``` 可用于 ```pretty print```。这些格式化器可按类型自定义（后续会介绍）
```rust
    fn main() {
        let a = [
            [40, 0], // Define a nested array
            [41, 0],
            [42, 1],
        ];
        for x in a {
            println!("{x:?}");
        }
    }
```
----
### Rust 元组 {#rust-tuples}
- 元组大小固定，可将任意类型组合成单一复合类型
    - 各成员类型可通过相对位置索引（`.0`、`.1`、`.2`、……）。空元组 `()` 称为 unit 值，相当于 void 返回值
    - Rust 支持元组解构，便于将变量绑定到各个元素
```rust
fn get_tuple() -> (u32, bool) {
    (42, true)        
}

fn main() {
   let t : (u8, bool) = (42, true);
   let u : (u32, bool) = (43, false);
   println!("{}, {}", t.0, t.1);
   println!("{}, {}", u.0, u.1);
   let (num, flag) = get_tuple(); // Tuple destructuring
   println!("{num}, {flag}");
}
```

### Rust 引用 {#rust-references}
- Rust 中的引用大致相当于 C 中的指针，但有关键差异
    - 在任意时刻，对同一变量可以有任意数量的只读（不可变）引用。引用不能超出变量作用域（这是称为**生命周期**的核心概念；后续详述）
    - 对可变变量只允许一个可写（可变）引用，且不得与其他任何引用重叠。
```rust
fn main() {
    let mut a = 42;
    {
        let b = &a;
        let c = b;
        println!("{} {}", *b, *c); // The compiler automatically dereferences *c
        
        let d = &mut a;
        
        /*
         * Uncommenting the line below would cause the
         * program to not compile, because `b` is used
         * while the mutable reference `d` is live in the current scope
         * 
         * You cannot have a mutable and immutable reference in use in the same scope
         * at the same time!
         */
        // println!("{}", *b);
    }
    let d = &mut a; // Ok: b and c are not in scope
    *d = 43;
}
```

----
# Rust 切片 {#rust-slices}
- Rust 引用可用于创建数组的子集
    - 与编译期长度固定的数组不同，切片可以是任意大小。内部实现上，切片是包含长度与指向原数组首元素指针的「胖指针」
```rust
fn main() {
    let a = [40, 41, 42, 43];
    let b = &a[1..a.len()]; // A slice starting with the second element in the original
    let c = &a[1..]; // Same as the above
    let d = &a[..]; // Same as &a[0..] or &a[0..a.len()]
    println!("{b:?} {c:?} {d:?}");
}
```
----
# Rust 常量与 static {#rust-constants-and-statics}
- ```const``` 关键字用于定义常量。常量在**编译期**求值并内联到程序中
- ```static``` 关键字用于定义类似 C/C++ 全局变量的等价物。static 变量有可寻址内存位置，创建一次并在程序整个生命周期内存在
```rust
const SECRET_OF_LIFE: u32 = 42;
static GLOBAL_VARIABLE : u32 = 2;
fn main() {
    println!("The secret of life is {}", SECRET_OF_LIFE);
    println!("Value of global variable is {GLOBAL_VARIABLE}")
}
```

----
# Rust 字符串：`String` 与 `&str` {#rust-strings-string-vs-str}

- Rust 有**两种**用途不同的字符串类型
    - `String` — 拥有所有权、堆分配、可增长（类似 C 的 `malloc` 缓冲区，或 C++ 的 `std::string`）
    - `&str` — 借用、轻量引用（类似带长度的 C `const char*`，或 C++ 的 `std::string_view`——但 `&str` 经**生命周期**检查，不会悬垂）
    - 与 C 以 null 结尾的字符串不同，Rust 字符串跟踪长度并保证为有效 UTF-8

> **面向 C++ 开发者：** `String` ≈ `std::string`，`&str` ≈ `std::string_view`。与 `std::string_view` 不同，`&str` 在其整个生命周期内由借用检查器保证有效。

## `String` 与 `&str`：拥有 vs 借用

> **生产实践：** 参见 [JSON 处理：nlohmann::json → serde](ch17-2-avoiding-unchecked-indexing.md#json-handling-nlohmannjson--serde)，了解生产代码中 serde 与字符串处理的配合。

| **方面** | **C `char*`** | **C++ `std::string`** | **Rust `String`** | **Rust `&str`** |
|------------|--------------|----------------------|-------------------|----------------|
| **内存** | 手动（`malloc`/`free`） | 堆分配，拥有缓冲区 | 堆分配，自动释放 | 借用引用（经生命周期检查） |
| **可变性** | 指针始终可变 | 可变 | 需 `mut` 才可变 | 始终不可变 |
| **大小信息** | 无（依赖 `'\0'`） | 跟踪长度与容量 | 跟踪长度与容量 | 跟踪长度（胖指针） |
| **编码** | 未指定（通常 ASCII） | 未指定（通常 ASCII） | 保证有效 UTF-8 | 保证有效 UTF-8 |
| **Null 终止符** | 需要 | 需要（`c_str()`） | 不使用 | 不使用 |

```rust
fn main() {
    // &str - string slice (borrowed, immutable, usually a string literal)
    let greeting: &str = "Hello";  // Points to read-only memory

    // String - owned, heap-allocated, growable
    let mut owned = String::from(greeting);  // Copies data to heap
    owned.push_str(", World!");        // Grow the string
    owned.push('!');                   // Append a single character

    // Converting between String and &str
    let slice: &str = &owned;          // String -> &str (free, just a borrow)
    let owned2: String = slice.to_string();  // &str -> String (allocates)
    let owned3: String = String::from(slice); // Same as above

    // String concatenation (note: + consumes the left operand)
    let hello = String::from("Hello");
    let world = String::from(", World!");
    let combined = hello + &world;  // hello is moved (consumed), world is borrowed
    // println!("{hello}");  // Won't compile: hello was moved

    // Use format! to avoid move issues
    let a = String::from("Hello");
    let b = String::from("World");
    let combined = format!("{a}, {b}!");  // Neither a nor b is consumed

    println!("{combined}");
}
```

## 为何不能用 `[]` 索引字符串
```rust
fn main() {
    let s = String::from("hello");
    // let c = s[0];  // Won't compile! Rust strings are UTF-8, not byte arrays

    // Safe alternatives:
    let first_char = s.chars().next();           // Option<char>: Some('h')
    let as_bytes = s.as_bytes();                 // &[u8]: raw UTF-8 bytes
    let substring = &s[0..1];                    // &str: "h" (byte range, must be valid UTF-8 boundary)

    println!("First char: {:?}", first_char);
    println!("Bytes: {:?}", &as_bytes[..5]);
}
```

## 练习：字符串操作

🟢 **入门**
- 编写函数 `fn count_words(text: &str) -> usize`，统计字符串中由空白分隔的单词数
- 编写函数 `fn longest_word(text: &str) -> &str`，返回最长单词（提示：需考虑生命周期——为何返回类型是 `&str` 而非 `String`？）

<details><summary>Solution (click to expand)</summary>

```rust
fn count_words(text: &str) -> usize {
    text.split_whitespace().count()
}

fn longest_word(text: &str) -> &str {
    text.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

fn main() {
    let text = "the quick brown fox jumps over the lazy dog";
    println!("Word count: {}", count_words(text));       // 9
    println!("Longest word: {}", longest_word(text));     // "jumps"
}
```

</details>

# Rust 结构体 {#rust-structs}
- ```struct``` 关键字声明用户定义的结构体类型
    - ```struct``` 成员可以具名，也可以匿名（元组结构体）
- 与 C++ 等语言不同，Rust 没有「数据继承」概念
```rust
fn main() {
    struct MyStruct {
        num: u32,
        is_secret_of_life: bool,
    }
    let x = MyStruct {
        num: 42,
        is_secret_of_life: true,
    };
    let y = MyStruct {
        num: x.num,
        is_secret_of_life: x.is_secret_of_life,
    };
    let z = MyStruct { num: x.num, ..x }; // The .. means copy remaining
    println!("{} {} {}", x.num, y.is_secret_of_life, z.num);
}
```

# Rust 元组结构体
- Rust 元组结构体类似元组，各字段没有名称
    - 与元组一样，用 `.0`、`.1`、`.2`、…… 访问各元素。常见用途是用元组结构体包装基本类型以创建自定义类型。**这有助于避免混淆同类型的不同值**
```rust
struct WeightInGrams(u32);
struct WeightInMilligrams(u32);
fn to_weight_in_grams(kilograms: u32) -> WeightInGrams {
    WeightInGrams(kilograms * 1000)
}

fn to_weight_in_milligrams(w : WeightInGrams) -> WeightInMilligrams  {
    WeightInMilligrams(w.0 * 1000)
}

fn main() {
    let x = to_weight_in_grams(42);
    let y = to_weight_in_milligrams(x);
    // let z : WeightInGrams = x;  // Won't compile: x was moved into to_weight_in_milligrams()
    // let a : WeightInGrams = y;   // Won't compile: type mismatch (WeightInMilligrams vs WeightInGrams)
}
```


**注意**：`#[derive(...)]` 属性会为结构体和枚举自动生成常见 Trait 实现。本课程中会频繁使用：
```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);           // Debug: works because of #[derive(Debug)]
    let p2 = p.clone();           // Clone: works because of #[derive(Clone)]
    assert_eq!(p, p2);            // PartialEq: works because of #[derive(PartialEq)]
}
```
Trait 系统后续会深入讲解，但 `#[derive(Debug)]` 非常实用，几乎应对每个 `struct` 和 `enum` 都加上。

# Rust `Vec` 类型 {#rust-vec-type}
- ```Vec<T>``` 类型实现动态堆分配缓冲区（类似 C 中手动管理的 `malloc`/`realloc` 数组，或 C++ 的 `std::vector`）
    - 与固定大小数组不同，`Vec` 可在运行时增长与收缩
    - `Vec` 拥有其数据并自动管理内存分配/释放
- 常见操作：`push()`、`pop()`、`insert()`、`remove()`、`len()`、`capacity()`
```rust
fn main() {
    let mut v = Vec::new();    // Empty vector, type inferred from usage
    v.push(42);                // Add element to end - Vec<i32>
    v.push(43);                
    
    // Safe iteration (preferred)
    for x in &v {              // Borrow elements, don't consume vector
        println!("{x}");
    }
    
    // Initialization shortcuts
    let mut v2 = vec![1, 2, 3, 4, 5];           // Macro for initialization
    let v3 = vec![0; 10];                       // 10 zeros
    
    // Safe access methods (preferred over indexing)
    match v2.get(0) {
        Some(first) => println!("First: {first}"),
        None => println!("Empty vector"),
    }
    
    // Useful methods
    println!("Length: {}, Capacity: {}", v2.len(), v2.capacity());
    if let Some(last) = v2.pop() {             // Remove and return last element
        println!("Popped: {last}");
    }
    
    // Dangerous: direct indexing (can panic!)
    // println!("{}", v2[100]);  // Would panic at runtime
}
```
> **生产实践：** 参见 [避免未检查索引](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing)，了解生产 Rust 代码中安全的 `.get()` 模式。

# Rust `HashMap` 类型 {#rust-hashmap-type}
- ```HashMap``` 实现泛型 ```key``` -> ```value``` 查找（亦称 ```dictionary``` 或 ```map```）
```rust
fn main() {
    use std::collections::HashMap;  // Need explicit import, unlike Vec
    let mut map = HashMap::new();       // Allocate an empty HashMap
    map.insert(40, false);  // Type is inferred as int -> bool
    map.insert(41, false);
    map.insert(42, true);
    for (key, value) in map {
        println!("{key} {value}");
    }
    let map = HashMap::from([(40, false), (41, false), (42, true)]);
    if let Some(x) = map.get(&43) {
        println!("43 was mapped to {x:?}");
    } else {
        println!("No mapping was found for 43");
    }
    let x = map.get(&43).or(Some(&false));  // Default value if key isn't found
    println!("{x:?}"); 
}
```

# 练习：`Vec` 与 `HashMap` {#exercise-vec-and-hashmap}

🟢 **入门**
- 创建包含若干条目的 ```HashMap<u32, bool>```（确保部分值为 ```true```、部分为 ```false```）。遍历 hashmap 中所有元素，将键放入一个 ```Vec```，值放入另一个

<details><summary>Solution (click to expand)</summary>

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::from([(1, true), (2, false), (3, true), (4, false)]);
    let mut keys = Vec::new();
    let mut values = Vec::new();
    for (k, v) in &map {
        keys.push(*k);
        values.push(*v);
    }
    println!("Keys:   {keys:?}");
    println!("Values: {values:?}");

    // Alternative: use iterators with unzip()
    let (keys2, values2): (Vec<u32>, Vec<bool>) = map.into_iter().unzip();
    println!("Keys (unzip):   {keys2:?}");
    println!("Values (unzip): {values2:?}");
}
```

</details>

---

## 深入：C++ 引用 vs Rust 引用 {#c-references-vs-rust-references--key-differences}

> **面向 C++ 开发者：** C++ 程序员常假设 Rust 的 `&T` 与 C++ 的 `T&` 类似。表面相似，但存在根本差异，容易混淆。C 开发者可跳过本节——Rust 引用在 [所有权与借用](ch07-ownership-and-borrowing.md) 中已有讲解。

#### 1. 无右值引用与万能引用

在 C++ 中，`&&` 依上下文有两种含义：

```cpp
// C++: && means different things:
int&& rref = 42;           // Rvalue reference — binds to temporaries
void process(Widget&& w);   // Rvalue reference — caller must std::move

// Universal (forwarding) reference — deduced template context:
template<typename T>
void forward(T&& arg) {     // NOT an rvalue ref! Deduced as T& or T&&
    inner(std::forward<T>(arg));  // Perfect forwarding
}
```

**Rust 中不存在这些。** `&&` 只是逻辑与运算符。

```rust
// Rust: && is just boolean AND
let a = true && false; // false

// Rust has NO rvalue references, no universal references, no perfect forwarding.
// Instead:
//   - Move is the default for non-Copy types (no std::move needed)
//   - Generics + trait bounds replace universal references
//   - No temporary-binding distinction — values are values

fn process(w: Widget) { }      // Takes ownership (like C++ value param + implicit move)
fn process_ref(w: &Widget) { } // Borrows immutably (like C++ const T&)
fn process_mut(w: &mut Widget) { } // Borrows mutably (like C++ T&, but exclusive)
```

| C++ 概念 | Rust 等价 | 说明 |
|-------------|-----------------|-------|
| `T&`（左值引用） | `&T` 或 `&mut T` | Rust 分为共享与独占 |
| `T&&`（右值引用） | 直接用 `T` | 按值接收 = 取得所有权 |
| 模板中的 `T&&`（万能引用） | `impl Trait` 或 `<T: Trait>` | 泛型替代转发 |
| `std::move(x)` | `x`（直接使用） | 移动是默认行为 |
| `std::forward<T>(x)` | 无需等价物 | 没有万能引用可转发 |

#### 2. 移动是位拷贝——无移动构造函数

在 C++ 中，移动是*用户定义*操作（移动构造/移动赋值）。在 Rust 中，移动始终是值的**按位 memcpy**，源被作废：

```rust
// Rust move = memcpy the bytes, mark source as invalid
let s1 = String::from("hello");
let s2 = s1; // Bytes of s1 are copied to s2's stack slot
              // s1 is now invalid — compiler enforces this
// println!("{s1}"); // ❌ Compile error: value used after move
```

```cpp
// C++ move = call the move constructor (user-defined!)
std::string s1 = "hello";
std::string s2 = std::move(s1); // Calls string's move ctor
// s1 is now a "valid but unspecified state" zombie
std::cout << s1; // Compiles! Prints... something (empty string, usually)
```

**后果**：
- Rust 无需 Rule of Five（无需定义拷贝构造、移动构造、拷贝赋值、移动赋值、析构）
- 没有移动后「僵尸」状态——编译器直接禁止访问
- 移动无需考虑 `noexcept`——按位拷贝不会抛异常

#### 3. 自动解引用：编译器穿透间接层

Rust 通过 `Deref` Trait 自动解引用多层指针/包装。C++ 无直接等价物：

```rust
use std::sync::{Arc, Mutex};

// Nested wrapping: Arc<Mutex<Vec<String>>>
let data = Arc::new(Mutex::new(vec!["hello".to_string()]));

// In C++, you'd need explicit unlocking and manual dereferencing at each layer.
// In Rust, the compiler auto-derefs through Arc → Mutex → MutexGuard → Vec:
let guard = data.lock().unwrap(); // Arc auto-derefs to Mutex
let first: &str = &guard[0];      // MutexGuard→Vec (Deref), Vec[0] (Index),
                                   // &String→&str (Deref coercion)
println!("First: {first}");

// Method calls also auto-deref:
let boxed_string = Box::new(String::from("hello"));
println!("Length: {}", boxed_string.len());  // Box→String, then String::len()
// No need for (*boxed_string).len() or boxed_string->len()
```

**Deref 强制转换**也适用于函数参数——编译器插入解引用使类型匹配：

```rust
fn greet(name: &str) {
    println!("Hello, {name}");
}

fn main() {
    let owned = String::from("Alice");
    let boxed = Box::new(String::from("Bob"));
    let arced = std::sync::Arc::new(String::from("Carol"));

    greet(&owned);  // &String → &str  (1 deref coercion)
    greet(&boxed);  // &Box<String> → &String → &str  (2 deref coercions)
    greet(&arced);  // &Arc<String> → &String → &str  (2 deref coercions)
    greet("Dave");  // &str already — no coercion needed
}
// In C++ you'd need .c_str() or explicit conversions for each case.
```

**Deref 链**：调用 `x.method()` 时，Rust 的方法解析先尝试接收者类型 `T`，再 `&T`、`&mut T`。若无匹配，则通过 `Deref` Trait 解引用并对目标类型重复。可穿透多层——因此 `Box<Vec<T>>` 能像 `Vec<T>` 一样「直接可用」。Deref *强制转换*（用于函数参数）是相关但独立的机制，将 `&Box<String>` 自动转为 `&str`，通过链式 `Deref` 实现。

#### 4. 无空引用，无可选引用

```cpp
// C++: references can't be null, but pointers can, and the distinction is blurry
Widget& ref = *ptr;  // If ptr is null → UB
Widget* opt = nullptr;  // "optional" reference via pointer
```

```rust
// Rust: references are ALWAYS valid — guaranteed by the borrow checker
// No way to create a null or dangling reference in safe code
let r: &i32 = &42; // Always valid

// "Optional reference" is explicit:
let opt: Option<&Widget> = None; // Clear intent, no null pointer
if let Some(w) = opt {
    w.do_something(); // Only reachable when present
}
```

#### 5. 引用不能重新绑定目标

```cpp
// C++: a reference is an alias — it can't be rebound
int a = 1, b = 2;
int& r = a;
r = b;  // This ASSIGNS b's value to a — it does NOT rebind r!
// a is now 2, r still refers to a
```

```rust
// Rust: let bindings can shadow, but references follow different rules
let a = 1;
let b = 2;
let r = &a;
// r = &b;   // ❌ Cannot assign to immutable variable
let r = &b;  // ✅ But you can SHADOW r with a new binding
             // The old binding is gone, not reseated

// With mut:
let mut r = &a;
r = &b;      // ✅ r now points to b — this IS rebinding (not assignment through)
```

> **心智模型**：在 C++ 中，引用是某一对象的永久别名。
> 在 Rust 中，引用是带生命周期保证的指针值，
> 遵循普通变量绑定规则——默认可变需 `mut` 才能重新绑定。
