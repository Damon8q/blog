---
title: "[Rust course]-16 高级进阶-深入类型"
date: 2022-05-30T10:57:00+08:00
lastmod: 2022-05-30T10:57:00+08:00
author: nange
draft: false
description: "Rust深入类型"

categories: ["rust"]
series: ["rust-course"]
series_weight: 17
tags: ["rust-notes"]
---

Rust 的学习难度之恶名，可能有一半来源于 Rust 的类型系统。本章学习如何创建自定义类型，以及了解何为动态大小的类型。

## newtype

何为 `newtype`？简单来说，就是使用[元组结构体](https://nange.github.io/posts/2022/05/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E5%A4%8D%E5%90%88%E7%B1%BB%E5%9E%8B/#%E5%85%83%E7%BB%84%E7%BB%93%E6%9E%84%E4%BD%93tuple-struct)的方式将已有的类型包裹起来：`struct Meters(u32);`，那么此处 `Meters` 就是一个 `newtype`。

为何需要 `newtype`？

- 自定义类型可以让我们给出更有意义和可读性的类型名，例如与其使用 `u32` 作为距离的单位类型，我们可以使用 `Meters`，它的可读性要好得多
- 对于某些场景，只有 `newtype` 可以很好地解决
- 隐藏内部类型的细节

从第二点说起。

### 为外部类型实现外部特征

在前面章节有讲过，如果在外部类型上实现外部特征必须使用 `newtype` 的方式，否则你就得遵循孤儿规则：要为类型 `A` 实现特征 `T`，那么 `A` 或者 `T` 必须至少有一个在当前的作用范围内。

例如，如果想使用 `println!("{}", v)` 的方式去格式化输出一个动态数组 `Vec`，以期给用户提供更加清晰可读的内容，那么就需要为 `Vec` 实现 `Display` 特征，但是这里有一个问题： `Vec` 类型定义在标准库中，`Display` 亦然，这时就可以祭出大杀器 `newtype` 来解决：

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

这样就实现了对 `Vec` 动态数组的格式化输出。

### 更好的可读性及类型异化

示例：

```rust
use std::ops::Add;
use std::fmt;

struct Meters(u32);
impl fmt::Display for Meters {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "目标地点距离你{}米", self.0)
    }
}

impl Add for Meters {
    type Output = Self;

    fn add(self, other: Meters) -> Self {
        Self(self.0 + other.0)
    }
}
fn main() {
    let d = calculate_distance(Meters(10), Meters(20));
    println!("{}", d);
}

fn calculate_distance(d1: Meters, d2: Meters) -> Meters {
    d1 + d2
}
```

上面代码创建了一个 `newtype` Meters，为其实现 `Display` 和 `Add` 特征，接着对两个距离进行求和计算，最终打印出该距离：

```rust
目标地点距离你30米
```

除了可读性外，还有一个极大的优点：如果给 `calculate_distance` 传一个其它的类型，例如 `struct MilliMeters(u32);`，该代码将无法编译。尽管 `Meters` 和 `MilliMeters` 都是对 `u32` 类型的简单包装，但是**它们是不同的类型**！

### 隐藏内部类型的细节

Rust 的类型有很多自定义的方法，假如我们把某个类型传给了用户，但是又不想用户调用这些方法，就可以使用 `newtype`：

```rust
struct Meters(u32);

fn main() {
    let i: u32 = 2;
    assert_eq!(i.pow(2), 4);

    let n = Meters(i);
    // 下面的代码将报错，因为`Meters`类型上没有`pow`方法
    // assert_eq!(n.pow(2), 4);
}
```

不过这种方式实际上也是掩耳盗铃，因为用户依然可以通过 `n.0.pow(2)` 的方式来调用内部类型的方法 :)

## 类型别名(Type Alias)

除了使用 `newtype`，我们还可以使用一个更传统的方式来创建新类型：类型别名

```rust
type Meters = u32
```

类型别名的方式看起来比 `newtype` 顺眼的多，而且跟其它语言的使用方式几乎一致，但是： **类型别名并不是一个独立的全新的类型，而是某一个类型的别名**，因此编译器依然会把 `Meters` 当 `u32` 来使用：

```rust
type Meters = u32;

let x: u32 = 5;
let y: Meters = 5;

println!("x + y = {}", x + y);
```

上面的代码将顺利编译通过，但是如果你使用 `newtype` 模式，该代码将无情报错，简单做个总结：

- 类型别名仅仅是别名，只是为了让可读性更好，并不是全新的类型，`newtype` 才是！
- 类型别名无法实现*为外部类型实现外部特征*等功能，而 `newtype` 可以

类型别名除了让类型可读性更好，还能**减少模版代码的使用**：

```rust
let f: Box<dyn Fn() + Send + 'static> = Box::new(|| println!("hi"));

fn takes_long_type(f: Box<dyn Fn() + Send + 'static>) {
    // --snip--
}

fn returns_long_type() -> Box<dyn Fn() + Send + 'static> {
    // --snip--
}
```

因为 `f` 的类型贼长，导致了后面我们在使用它时，到处都充斥这些不太优美的类型标注，好在类型别名可简化此问题：

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;

let f: Thunk = Box::new(|| println!("hi"));

fn takes_long_type(f: Thunk) {
    // --snip--
}

fn returns_long_type() -> Thunk {
    // --snip--
}
```

在标准库中，类型别名应用最广的就是简化 `Result<T, E>` 枚举。

由于使用 `std::io` 库时，它的所有错误类型都是 `std::io::Error`，那么我们完全可以把该错误对用户隐藏起来，只在内部使用即可，因此就可以使用类型别名来简化实现：

```rust
type Result<T> = std::result::Result<T, std::io::Error>;
```

这样一来，其它库只需要使用 `std::io::Result<T>` 即可替代冗长的 `std::result::Result<T, std::io::Error>` 类型。

更香的是，由于它只是别名，因此我们可以用它来调用真实类型的所有方法，包括 `?` 符号！

## ! 永不返回类型

在[函数](https://nange.github.io/posts/2022/04/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E5%9F%BA%E6%9C%AC%E7%B1%BB%E5%9E%8B/#%E5%87%BD%E6%95%B0)那章，曾经介绍过 `!` 类型：`!` 用来说明一个函数永不返回任何值：

```
fn main() {
    let i = 2;
    let v = match i {
       0..=3 => i,
       _ => println!("不合规定的值:{}", i)
    };
}
```

上面函数，会报出一个编译错误:

```rust
error[E0308]: `match` arms have incompatible types // match的分支类型不同
 --> src/main.rs:5:13
  |
3 |       let v = match i {
  |  _____________-
4 | |        0..3 => i,
  | |                - this is found to be of type `{integer}` // 该分支返回整数类型
5 | |        _ => println!("不合规定的值:{}", i)
  | |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected integer, found `()` // 该分支返回()单元类型
6 | |     };
  | |_____- `match` arms have incompatible types
```

既然 `println` 不行，那再试试 `panic`:

```rust
fn main() {
    let i = 2;
    let v = match i {
       0..=3 => i,
       _ => panic!("不合规定的值:{}", i)
    };
}
```

此处 `panic` 竟然通过了编译。原因是：`panic的返回值是!`，代表它不会返回任何值，既然没有任何返回值，那自然不会存在分支类型不匹配的情况。













