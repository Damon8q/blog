---
title: "[Rust course]-22 高级进阶-错误处理"
date: 2022-10-10T10:20:00+08:00
lastmod: 2022-10-11T11:05:00+08:00
author: nange
draft: false
description: "Rust错误处理"

categories: ["programming"]
series: ["rust-course"]
series_weight: 22
tags: ["rust"]
---

Rust是通过`Result`返回结果，用于错误处理，`?`用于错误传播。本章继续学习如何对`Result`做进一步处理，以及如何定义自己的错误类型。

## 组合器

组合器用来对返回结果进行判断或变换。

### or() 和 and()

- `or()`，表达式按照顺序求值，若任何一个表达式的结果是 `Some` 或 `Ok`，则该值会立刻返回当前表达式值
- `and()`，若两个表达式的结果都是 `Some` 或 `Ok`，则**第二个表达式中的值被返回**。若任何一个的结果是 `None` 或 `Err` ，则立刻返回`None`或`Err`

```rust
fn main() {
  let s1 = Some("some1");
  let s2 = Some("some2");
  let n: Option<&str> = None;

  let o1: Result<&str, &str> = Ok("ok1");
  let o2: Result<&str, &str> = Ok("ok2");
  let e1: Result<&str, &str> = Err("error1");
  let e2: Result<&str, &str> = Err("error2");

  assert_eq!(s1.or(s2), s1); // Some1 or Some2 = Some1
  assert_eq!(s1.or(n), s1);  // Some or None = Some
  assert_eq!(n.or(s1), s1);  // None or Some = Some
  assert_eq!(n.or(n), n);    // None1 or None2 = None2

  assert_eq!(o1.or(o2), o1); // Ok1 or Ok2 = Ok1
  assert_eq!(o1.or(e1), o1); // Ok or Err = Ok
  assert_eq!(e1.or(o1), o1); // Err or Ok = Ok
  assert_eq!(e1.or(e2), e2); // Err1 or Err2 = Err2

  assert_eq!(s1.and(s2), s2); // Some1 and Some2 = Some2
  assert_eq!(s1.and(n), n);   // Some and None = None
  assert_eq!(n.and(s1), n);   // None and Some = None
  assert_eq!(n.and(n), n);    // None1 and None2 = None1

  assert_eq!(o1.and(o2), o2); // Ok1 and Ok2 = Ok2
  assert_eq!(o1.and(e1), e1); // Ok and Err = Err
  assert_eq!(e1.and(o1), e1); // Err and Ok = Err
  assert_eq!(e1.and(e2), e1); // Err1 and Err2 = Err1
}
```

### or_else 和 and_then

`or_else()`和`and_then()`，跟 `or()` 和 `and()` 类似，区别在于：

* 它们接收的是一个闭包
* `or_else`闭包，没有参数；而`and_then`闭包有一个参数，这个参数即是前一个调用的值

```rust
fn main() {
    let s1 = Some("some1");
    let s2 = Some("some2");

    assert_eq!(s1.or_else(|| None), s1);
    assert_eq!(
        s1.and_then(|s| {
            if s.eq("some1") {
                return Some(s);
            }
            None
        }),
        s1
    );
    assert_eq!(
        s2.and_then(|s| {
            if s.eq("some1") {
                return Some(s);
            }
            None
        }),
        None
    );
}
```

### filter()

`filter` 用于对 `Option` 进行过滤：

```rust
fn main() {
    let s1 = Some(3);
    let s2 = Some(6);

    let fn_is_even = |x: &i8| x % 2 == 0;

    assert_eq!(s1.filter(fn_is_even), None);
    assert_eq!(s2.filter(fn_is_even), s2);
    assert_eq!(None.filter(fn_is_even), None);
}
```

### map() 和 map_err()

`map` 可以将 `Some` 或 `Ok` 中的值映射为另一个：

```rust
fn main() {
    let s1 = Some("abcdef");
    assert_eq!(s1.map(|s| s.chars().count()), Some(6));

    let n1: Option<&str> = None;
    assert_eq!(n1.map(|s| s.chars().count()), None);

    let o1: Result<&str, &str> = Ok("abcdef");
    assert_eq!(o1.map(|s| s.chars().count()), Ok(6));

    let e1: Result<&str, &str> = Err("123");
    assert_eq!(e1.map_err(|s| s.parse().unwrap()), Err(123));

    // 如果不是Err类型，map_err直接返回原值，即透传
    assert_eq!(o1.map_err(|s| s.chars().count()), Ok("abcdef"));
}
```

### map_or() 和 map_or_else()

`map_or` 在 `map` 的基础上提供了一个默认值，而`map_or_else` 与 `map_or` 类似，但是它是通过一个闭包来提供默认值:

```rust
fn main() {
    const V_DEFAULT: i32 = 1;

    let s: Result<i32, ()> = Ok(10);
    assert_eq!(s.map_or(V_DEFAULT, |v| v + 2), 12);

    let e: Result<i32, ()> = Err(());
    assert_eq!(e.map_or(V_DEFAULT, |v| v + 2), 1);

    // Option的用法和Result同上

    let o: Result<i32, i32> = Ok(10);
    let e: Result<i32, i32> = Err(5);
    assert_eq!(o.map_or_else(|v| v + 1, |v| v + 2), 12);
    assert_eq!(e.map_or_else(|v| v + 1, |v| v + 2), 6);
}
```

### ok_or() 和 ok_or_else()

这两个方法可以将 `Option` 类型转换为 `Result` 类型。其中 `ok_or` 接收一个默认的 `Err` 参数， 而`ok_or_else`接收一个闭包作为`Err`参数：

```rust
fn main() {
    let s: Option<&str> = Some("abcd");
    let n: Option<&str> = None;

    assert_eq!(s.ok_or("error"), Ok("abcd"));
    assert_eq!(n.ok_or("error"), Err("error"));

    assert_eq!(s.ok_or_else(|| "error"), Ok("abcd"));
    assert_eq!(n.ok_or_else(|| "error"), Err("error"));
}
```

Rust标准库还有更多的API，需要时可进行查看。



## 自定义错误类型

在实际项目总有需要自定义错误类型的时候，因此Rust 在标准库中提供了一些可复用的特征，例如 `std::error::Error` 特征：

```rust
use std::fmt::{Debug, Display};

pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }
    ...
}
```

> 实际上，自定义错误类型只需要实现 `Debug` 和 `Display` 特征即可，`source` 方法是可选的，而 `Debug` 特征往往也无需手动实现，可以直接通过 `derive` 来派生。

定义一个具有错误码和错误信息的错误：

```rust
use std::fmt;

struct AppError {
    code: usize,
    message: String,
}

// 根据错误码显示不同的错误信息
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let err_msg = match self.code {
            404 => "Sorry, Can not find the Page!",
            _ => "Sorry, something is wrong! Please Try Again!",
        };

        write!(f, "{}", err_msg)
    }
}

impl fmt::Debug for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(
            f,
            "AppError {{ code: {}, message: {} }}",
            self.code, self.message
        )
    }
}

fn produce_error() -> Result<(), AppError> {
    Err(AppError {
        code: 404,
        message: String::from("Page not found"),
    })
}

fn main() {
    match produce_error() {
        Err(e) => eprintln!("{}", e), // 抱歉，未找到指定的页面!
        _ => println!("No error"),
    }

    eprintln!("{:?}", produce_error()); // Err(AppError { code: 404, message: Page not found })

    eprintln!("{:#?}", produce_error());
    // Err(
    //     AppError { code: 404, message: Page not found }
    // )
}
```

### 错误转换

错误这么多，有时候我们会需要把各种错误统一转换成一种。 Rust 为我们提供了 `std::convert::From` 特征:

```rust
pub trait From<T>: Sized {
  fn from(_: T) -> Self;
}
```

举例：

```rust
use std::fs::File;
use std::io;

#[derive(Debug)]
struct AppError {
    kind: String,    // 错误类型
    message: String, // 错误信息
}

// 为 AppError 实现 std::convert::From 特征，由于 From 包含在 std::prelude 中，因此可以直接简化引入。
// 实现 From<io::Error> 意味着我们可以将 io::Error 错误转换成自定义的 AppError 错误
impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError {
            kind: String::from("io"),
            message: error.to_string(),
        }
    }
}

fn main() -> Result<(), AppError> {
    let _file = File::open("nonexistent_file.txt")?;

    Ok(())
}

// --------------- 上述代码运行后输出 ---------------
Error: AppError { kind: "io", message: "No such file or directory (os error 2)" }

```

 `?` 可以将错误进行隐式的强制转换，这就是`?`的方便和强大。

我们除了给`io::Error`实现转换，还可以给任何类型的错误类型实现转换，只要实现对应的`from`函数即可。



## 归一化不同的错误类型

在实际项目中，不同方法可能返回不同类型的错误，但当在一个函数内部调用了多个其他函数并且它们返回的错误类型不同时，我们该如何处理呢？一种处理方式是针对每种错误就地处理后，对外不再抛出错误，但实际上往往不能这样做，因为太麻烦了，也不一定符合实际要求，有可能就是要抛出错误才行，在这个函数层次可能根本不知道该如何处理。还有没有更好的办法呢？

还有三种方法：

- 使用特征对象 `Box<dyn Error>`
- 自定义错误类型
- 使用 `thiserror`

### Box<dyn Error>

只要实现了 `Debug + Display`特征，都可以转化为`dyn Error`

```rust
use std::fs::read_to_string;
use std::error::Error;
fn main() -> Result<(), Box<dyn Error>> {
  let html = render()?;
  println!("{}", html);
  Ok(())
}

fn render() -> Result<String, Box<dyn Error>> {
  let file = std::env::var("MARKDOWN")?;
  let source = read_to_string(file)?;
  Ok(source)
}

```

`Result` 实际上不会限制错误的类型，也就是一个类型就算不实现 `Error` 特征，它依然可以在 `Result<T, E>` 中作为 `E` 来使用。

### 自定义错误类型

与特征对象相比，自定义错误类型麻烦归麻烦，但是它非常灵活：

```rust
use std::fs::read_to_string;

fn main() -> Result<(), MyError> {
  let html = render()?;
  println!("{}", html);
  Ok(())
}

fn render() -> Result<String, MyError> {
  let file = std::env::var("MARKDOWN")?;
  let source = read_to_string(file)?;
  Ok(source)
}

#[derive(Debug)]
enum MyError {
  EnvironmentVariableNotFound,
  IOError(std::io::Error),
}

impl From<std::env::VarError> for MyError {
  fn from(_: std::env::VarError) -> Self {
    Self::EnvironmentVariableNotFound
  }
}

impl From<std::io::Error> for MyError {
  fn from(value: std::io::Error) -> Self {
    Self::IOError(value)
  }
}

impl std::error::Error for MyError {}

impl std::fmt::Display for MyError {
  fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
    match self {
      MyError::EnvironmentVariableNotFound => write!(f, "Environment variable not found"),
      MyError::IOError(err) => write!(f, "IO Error: {}", err.to_string()),
    }
  }
}

```

### 简化错误处理

### thiserror

[`thiserror`](https://github.com/dtolnay/thiserror)可以帮助我们简化上面的第二种解决方案：

```rust
use std::fs::read_to_string;

fn main() -> Result<(), MyError> {
  let html = render()?;
  println!("{}", html);
  Ok(())
}

fn render() -> Result<String, MyError> {
  let file = std::env::var("MARKDOWN")?;
  let source = read_to_string(file)?;
  Ok(source)
}

#[derive(thiserror::Error, Debug)]
enum MyError {
  #[error("Environment variable not found")]
  EnvironmentVariableNotFound(#[from] std::env::VarError),
  #[error(transparent)]
  IOError(#[from] std::io::Error),
}
```

如上所示，只要简单写写注释，就可以实现错误处理了。

### anyhow

[`anyhow`](https://github.com/dtolnay/anyhow) 和 `thiserror` 是同一个作者开发的，下面是作者关于选择两者的说明：

> 如果你想要设计自己的错误类型，同时给调用者提供具体的信息时，就使用 `thiserror`，例如当你在开发一个三方库代码时。如果你只想要简单，就使用 `anyhow`，例如在自己的应用服务中。

```rust
use std::fs::read_to_string;

use anyhow::Result;

fn main() -> Result<()> {
    let html = render()?;
    println!("{}", html);
    Ok(())
}

fn render() -> Result<String> {
    let file = std::env::var("MARKDOWN")?;
    let source = read_to_string(file)?;
    Ok(source)
}

```

