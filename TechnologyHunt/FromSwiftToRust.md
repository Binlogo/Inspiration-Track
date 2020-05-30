# From Swift to Rust

Rust 和 Swift 一样都是现代化的年轻编程语言，语言特性上都借鉴了很多不同语言的优点，语法上有诸多相似的地方，不过诞生的背景和目标不一样，所以也有差异很大的地方，有着不同的应用场景。

Swift 诞生在 Apple，2014 年发布 1.0 版本，在 WWDC 2015 开发者大会宣布开源，并于同年发布 2.0 版本。因此，Swift 最初的目标基本就是取代 Objective-C 成为苹果平台的下一代官方编程语言，在安全、快速这样的基础上，致力于在易学、易上手，以促进平台应用的进一步繁荣。之后的开源并推及到 Linux 平台和服务端，基本也是这个目标的延展。

Rust 诞生于 Mozilla，按 1.0 版本发布的话，比 Swift 还年轻 1 岁，2015 年发布的 1.0 版本，前不久过了个 [5 岁生日](https://twitter.com/rustlang/status/1261253754969640960)。Mozilla 开源背景和历史，以及在社区上的经验和积累是非常丰富的，因此 Rust 从最初就由良好的开源社区驱动，目标是成为安全可靠且高性能的系统级编程语言。既然是系统级语言，则对标的是 C 和 C++，无运行时和垃圾回收，确保性能的同时，能够方便地进行跨平台和嵌入式开发。

简单了解下背景，下面就通过对比下两门语言，从一个熟悉 Swift 的客户端开发视角来认识下 Rust 这位新朋友。

> 注：对于 Rust，我也正在学习中，文中如有错漏，还望指正，欢迎提 [issue](https://github.com/Binlogo/Inspiration-Track/issues)。

## 语言特性对比概览

- 变量绑定
  - 可变性
  - 作用域和遮蔽（shadowing）
- 类型系统
  - 类型推断（type inference）
  - 类型转换（cast）
  - 别名（alias）
- 值语义
  - 结构体
  - 枚举
- 引用语义
  - `Box<T>`
  - `Rc<T>`
  - `Arc<T>`
- 流程控制
  - 循环
  - 匹配
- 函数方法
  - 派发机制
  - 闭包
- 访问控制
- 内存管理
  - 可选型
- 泛型系统
- 分号的应用
- 并发编程

## 变量绑定

### 可变性

Rust 和 Swift 都默认推荐使用不可变性，并明确声明可变性。

```rust
// rust
let x = 666; // 默认不可变
// x = 999; <- 编译会报错

let mut y = 233; // let mut 明确声明可变性
y = 23333; // <- 编译通过
```

```swift
// swift
let x = 666
var y = 233
```

### 作用域和遮蔽（shadowing）

Rust 和 Swift 一样，变量绑定的作用域都被限定在一个代码块中。

```rust
// rust
fn main() {
  let long_lived_binding = 1; // 作用域为整个 main 函数
  
  {
    let short_lived_binding = 2; // 仅作用域当前代码块中
    println!("{}", short_lived_binding); // 打印 2
  }
  // 报错：cannot find value `short_lived_binding` in this scope
  println!("{}", short_lived_binding);
  
  println!("{}", long_lived_binding); // 打印 1
}
```

```swift
// swift
func main() {
    let longLivedBinding = 1

    do {
        let shortLivedBinding = 2
        print(shortLivedBinding)
    }
  	// 报错：Use of unresolved identifier 'shortLivedBinding'
    print(shortLivedBinding)

    print(longLivedBinding) // 打印 1
}
```

和 Swift 有所差别的是， Rust 还支持变量遮蔽，Swift 仅在可选型拆包时支持变量遮蔽。

```rust
// rust
let x = 1;
let x = 2; // 此时 x 的绑定会遮蔽上一个 `x` 的绑定
println!("{}", x); // 打印 2
```

```swift
// swift
let x = 1
let x = 2 // 报错：Definition conflicts with previous value

// 仅支持可选型的拆包“遮蔽”
let x: Int? = 1
print(x) // 打印：Optional(1)
if let x = x {
    print(x) // 打印 1， 此时 x 不再是可选型而是实际内容
}
```

## 类型系统

Rust 和 Swift 一样都是强类型的静态编程语言。因此，编译器能够在编译时检测许多内存安全性问题。

### 类型推断

Rust 和 Swift 都具备类型推导能力，也支持显示声明类型。

```rust
// rust
let x = 2.0; // 自动类型推导：f64
let y: f32 = 3.0; // 显示声明类型：f32
```

```swift
// swift
let x = 2.0 // Double
let y: Float = 3.0 // Float
```

### 类型转换（cast）

与 Swift 类似，Rust 不提供原生类型之间的隐式类型转换，而是使用 `as` 关键字进行显式转换。

```rust
// rust
let decimal = 3.14_f32;

// 不支持隐式类型转换，报错：mismatched types
let integer: u8 = decimal; 

let integer = decimal as u8; // 支持显式转换
```

```swift
// swift
let decimal = 3.14

// 不支持隐式类型转换，报错：Cannot convert value of type 'Double' to specified type 'Int'
let integer: Int = decimal
let integer = Int(decimal)
```

> 对于自定义类型转换，需要实现 [`From`](https://doc.rust-lang.org/std/convert/trait.From.html) 和 [`Into`](https://doc.rust-lang.org/std/convert/trait.Into.html) 这两个 trait。

### 别名（alias）

Swift 中使用 `typealias` 对类型进行别名，Rust 中使用 `type` 关键字，类型的名字须遵循驼峰命名，原生类型除外。

```rust
// rust
type NonaSecond = u64;
```

```swift
// swift
typealias NonaSecond = UInt64
```

## 值语义

Swift 中有结构体（struct）和枚举（enum）两种值语义的自定义类型，Rust 中也类似。

### 结构体

```rust
// rust

// 声明
struct Point {
    x: f32,
    y: f32,
}

// 使用`impl`关键字来实现方法
impl Point {
  fn new() -> Self {
    Self { x: 0.0, y, 0.0 }
  }
}

// 初始化
let point = Point { x: 0.5, y: 0.5 };
let oringin = Point::new();
// 支持解构
let Point { x: the_x, y: the_y } = point;
println!("{}, {}", the_x, the_y); // 打印：0.5, 0.5
```

```swift
// swift
struct Point {
    let x: Float
    let y: Float
}
let point = Point(x: 0.5, y: 0.5)
```

### 枚举

与 Swift 类似，Rust 中枚举也支持关联值

```rust
// rust
enum CustomError {
  CustomError1,
  CustomError2,
  CustomError3
}

// 关联值枚举
enum CustomError {
  CustomError1(&str),
  CustomError2(u32),
  CustomError3(&str, u32)
}
```

```swift
enum CustomError {
  case customError1
  case customError2
  case customError3
}

// 关联值枚举
enum CustomError {
  case customError1(String)
  case customError2(UInt)
  case customError3(String, UInt)
}
```

## 引用语义

Swift 中有类（class）来声明引用语义的自定义类型。但在 Rust 中没有类的概念，要使用引用语义，需要在值语义类型之上通过标准库中的 [`Box<T>`](https://doc.rust-lang.org/std/boxed/struct.Box.html)  、 [`Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html) 、[`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) 进行「封装」。

### Box

`Box<T>`是最简单的引用语义的封装，在堆上分配内存后的引用指针，且仅支持有一个所有者（owner，类似 Swift 中的强引用）。

```rust
// rust
let a = Box::new(5);
let b = a; // a 被`move`到了 b
// 报错：use of moved value: `a`
println!("{:?}", a);
```

### Rc（refence-counting）

相比 Box，`Rc<T>`  稍微更像 Swift 中的引用些了，通过「浅拷贝」支持多个引用了。

```rust
// rust
use std::rc::Rc;
let a = Rc::new([5]);
let b = a.clone(); // 相当于 Swift 中的浅拷贝
println!("{:?}", a); // 打印：[5]
```

### Arc（Atomically Reference Counted）

`Arc<T>` 直译一下的话，感觉有点熟悉，「自动引用计数」，但不要将它和 Swift 中内存管理方式混淆。Arc 和 Rc 的作用差不多，但 Rc 是在单线程下使用，Arc 是在多线程下使用，因此 `clone` 的实现是原子性的，因此能做到线程安全。

```rust
// rust
use std::sync::Arc;
let foo = Arc::new(vec![1.0, 2.0, 3.0]);
let a = foo.clone();
```

> Arc 是只读引用，要使用可写引用，需要再另外使用 `Mutex` 来实现，类似`Arc<Mutex<T>>`。

## 流程控制

### 循环

Rust 中循环提供 `loop`、`while`、`for`/`in` 三种方式，和 Swift 非常类似

- loop：无限循环，使用 `break` 退出或 `continue` 跳入下一个迭代

```rust
// rust
let mut count = 0;
loop {
  count += 1;
  if count == 3 {
    continue; // 跳过后续执行直接进入下个迭代
  }
  println!("{}", count);
  if count == 5 {
    break; // 退出循环
  }
}
```

```swift
// swift
var count = 0
repeat {
    count += 1
    if count == 3 {
        continue
    }
    print(count)
    if count == 5 {
        break
    }
} while true
```

Rust 中 loop 的嵌套是支持标签注明的，`break` 时默认退出当前循环，可以标识标签来直接退出外层循环。

```rust
'outer: loop {
    println!("Entered the outer loop");

    'inner: loop {
        println!("Entered the inner loop");

        // 默认只是中断内部的当前循环
        //break;

        // 直接会中断外层循环
        break 'outer;
    }

    println!("This point will never be reached");
}

println!("Exited the outer loop");
```

除此之外，loop 还支持通过 break 返回值。

```rust
let mut counter = 0;

let result = loop {
    counter += 1;

    if counter == 10 {
        break counter * 2;
    }
};

assert_eq!(result, 20);
```

- while：条件循环，和 Swift 以及其他许多语言的 while 使用无二致，就不示例了。

- forin：和 Swift 一样，支持区间和迭代的应用

```rust
// rust
```



### 匹配