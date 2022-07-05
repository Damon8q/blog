---
title: "[Rust course]-13 基础入门-注释和文档"
date: 2022-05-13T14:57:00+08:00
lastmod: 2022-05-13T14:57:00+08:00
author: nange
draft: false
description: "Rust注释和文档"

categories: ["编程语言"]
series: ["rust-course"]
series_weight: 13
tags: ["rust"]
---

## 注释的种类

Rust注释分为三类：

- 代码注释，用于说明某一块代码的功能，读者往往是同一个项目的协作开发者
- 文档注释，支持 `Markdown`，对项目描述、公共 API 等用户关心的功能进行介绍，同时还能提供示例代码，目标读者往往是想要了解你项目的人
- 包和模块注释，严格来说这也是文档注释中的一种，它主要用于说明当前包和模块的功能，方便用户迅速了解一个项目

## 代码注释

### 行注释

```rust
fn main() {
    // 我是Sun...
    // face
    let name = "sunface";
    let age = 18; // 今年好像是18岁
}
```

### 块注释

```rust
fn main() {
    /*
        我
        是
        S
        u
        n
        ... 淦，好长!
    */
    let name = "sunface";
    let age = "???"; // 今年其实。。。挺大了
}
```

只需要将注释内容使用 `/* */` 进行包裹即可。一般用行注释更多。

## 文档注释

当查看一个 `crates.io` 上的包时，往往需要通过它提供的文档来浏览相关的功能特性、使用方式，这种文档就是通过文档注释实现的。

Rust 提供了 `cargo doc` 的命令，可以用于把这些文档注释转换成 `HTML` 网页文件，最终展示给用户浏览，这样用户就知道这个包是做什么的以及该如何使用。

### 文档行注释

```rust
/// `add_one` 将指定值加1
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = my_crate::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

以上代码有几点需要注意：

- 文档注释需要位于 `lib` 类型的包中，例如 `src/lib.rs` 中
- 文档注释可以使用 `markdown`语法！例如 `# Examples` 的标题，以及代码块高亮
- 被注释的对象需要使用 `pub` 对外可见，记住：文档注释是给用户看的，**内部实现细节不应该被暴露出去**

### 文档块注释

````rust
/** `add_two` 将指定值加2


```
let arg = 5;
let answer = my_crate::add_two(arg);

assert_eq!(7, answer);
```
*/
pub fn add_two(x: i32) -> i32 {
    x + 2
}
````

通常习惯上使用行注释的情况更多。

### 查看文档 cargo doc

运行 `cargo doc` 可以直接生成 `HTML` 文件，放入*target/doc*目录下。

为了方便，我们使用 `cargo doc --open` 命令，可以在生成文档后，自动在浏览器中打开网页，最终效果如图所示：

### 常用文档标题

之前我们见到了在文档注释中该如何使用 `markdown`，其中包括 `# Examples` 标题。除了这个标题，还有一些常用的，可以在项目中酌情使用：

- **Panics**：函数可能会出现的异常状况，这样调用函数的人就可以提前规避
- **Errors**：描述可能出现的错误及什么情况会导致错误，有助于调用者针对不同的错误采取不同的处理方式
- **Safety**：如果函数使用 `unsafe` 代码，那么调用者就需要注意一些使用条件，以确保 `unsafe` 代码块的正常工作

话说回来，这些标题更多的是一种惯例，如果非要用中文标题也没问题，但是最好在团队中保持同样的风格 :)

## 包和模块级别注释

除了函数、结构体等 Rust 项的注释，还可以给包和模块添加注释，需要注意的是，**这些注释要添加到包、模块的最上方**！

与之前的任何注释一样，包级别的注释也分为两种：行注释 `//!` 和块注释 `/*! ... */`。

在 `src/lib.rs` 包根的最上方，添加：

```rust
/*! lib包是world_hello二进制包的依赖包，
 里面包含了compute等有用模块 */
//! lib包是world_hello二进制包的依赖包,
//! 里面包含了compute等有用模块.

pub mod compute;
```

再为该包根的子模块 `src/compute.rs` 添加注释：

```rust
//! 计算一些你口算算不出来的复杂算术题


/// `add_one`将指定值加1
///
```

## 文档测试

Rust 允许我们在文档注释中写单元测试用例：

```rust
/// `add_one` 将指定值加1
///
/// # Examples11
///
/// ```
/// let arg = 5;
/// let answer = world_hello::compute::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

以上的注释不仅仅是文档，还可以作为单元测试的用例运行，使用 `cargo test` 运行测试：

```rust
Doc-tests world_hello

running 2 tests
test src/compute.rs - compute::add_one (line 8) ... ok
test src/compute.rs - compute::add_two (line 22) ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.00s
```

#### 造成panic的文档测试

文档测试中的用例还可以造成 `panic`：

```rust
/// # Panics
///
/// The function panics if the second argument is zero.
///
/// ```rust
/// // panics on division by zero
/// world_hello::compute::div(10, 0);
/// ```
pub fn div(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    }

    a / b
}
```

以上测试运行后会 `panic`：

```rust
---- src/compute.rs - compute::div (line 38) stdout ----
Test executable failed (exit code 101).

stderr:
thread 'main' panicked at 'Divide-by-zero error', src/compute.rs:44:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

如果想要通过这种测试，可以添加 `should_panic`：

```rust
/// # Panics
///
/// The function panics if the second argument is zero.
///
/// ```rust,should_panic
/// // panics on division by zero
/// world_hello::compute::div(10, 0);
/// ```
```

通过 `should_panic`，告诉 Rust 我们这个用例会导致 `panic`，这样测试用例就能顺利通过。

#### 保留测试，隐藏文档

```rust
/// ```
/// # // 使用#开头的行会在文档中被隐藏起来，但是依然会在文档测试中运行
/// # fn try_main() -> Result<(), String> {
/// let res = world_hello::compute::try_div(10, 0)?;
/// # Ok(()) // returning from try_main
/// # }
/// # fn main() {
/// #    try_main().unwrap();
/// #
/// # }
/// ```
pub fn try_div(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("Divide-by-zero"))
    } else {
        Ok(a / b)
    }
}
```

以上文档注释中，我们使用 `#` 将不想让用户看到的内容隐藏起来，但是又不影响测试用例的运行，最终用户将只能看到那行没有隐藏的 `let res = world_hello::compute::try_div(10, 0)?;`。

## 文档注释中的代码跳转

Rust 在文档注释中还提供了一个非常强大的功能，那就是可以实现对外部项的链接：

### 跳转到标准库

```rust
/// `add_one` 返回一个[`Option`]类型
pub fn add_one(x: i32) -> Option<i32> {
    Some(x + 1)
}
```

此处的 **[`Option`]** 就是一个链接，指向了标准库中的 `Option` 枚举类型，有两种方式可以进行跳转:

- 在 IDE 中，使用 `Command + 鼠标左键`(macOS)，`CTRL + 鼠标左键`(Windows)
- 在文档中直接点击链接

还可以使用路径的方式跳转：

```rust
use std::sync::mpsc::Receiver;

/// [`Receiver<T>`]   [`std::future`].
///
///  [`std::future::Future`] [`Self::recv()`].
pub struct AsyncReceiver<T> {
    sender: Receiver<T>,
}

impl<T> AsyncReceiver<T> {
    pub async fn recv() -> T {
        unimplemented!()
    }
}
```

### 使用完整路径跳转到指定项

```rust
pub mod a {
    /// `add_one` 返回一个[`Option`]类型
    /// 跳转到[`crate::MySpecialFormatter`]
    pub fn add_one(x: i32) -> Option<i32> {
        Some(x + 1)
    }
}

pub struct MySpecialFormatter;
```

### 同名项的跳转

如果遇到同名项，可以使用标示类型的方式进行跳转：

```rust
/// 跳转到结构体  [`Foo`](struct@Foo)
pub struct Bar;

/// 跳转到同名函数 [`Foo`](fn@Foo)
pub struct Foo {}

/// 跳转到同名宏 [`foo!`]
pub fn Foo() {}

#[macro_export]
macro_rules! foo {
  () => {}
}
```

## 文档搜索别名

Rust 文档支持搜索功能，我们可以为自己的类型定义几个别名，以实现更好的搜索展现，当别名命中时，搜索结果会被放在第一位：

```rust
#[doc(alias = "x")]
#[doc(alias = "big")]
pub struct BigX;

#[doc(alias("y", "big"))]
pub struct BigY;
```

![img](/images/v2-1ab5b19d2bd06f3d83204d062b399bcd_1440w.png)

## 总结

在 Rust 中，注释分为三个主要类型：代码注释、文档注释、包和模块注释，每个注释类型都拥有两种形式：行注释和块注释，熟练掌握包模块和注释的知识，非常有助于我们创建工程性更强的项目。
