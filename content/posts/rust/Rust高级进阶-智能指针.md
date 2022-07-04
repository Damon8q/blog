---
title: "[Rust course]-18 高级进阶-智能指针"
date: 2022-06-01T11:35:00+08:00
lastmod: 2022-07-04T11:35:00+08:00
author: nange
draft: false
description: "Rust智能指针"

categories: ["rust"]
series: ["rust-course"]
series_weight: 18
tags: ["rust-notes"]
---

在各个编程语言中，指针的概念几乎都是相同的：**指针是一个包含了内存地址的变量，该内存地址引用或者指向了另外的数据**。

在 Rust 中，最常见的指针类型是引用，引用通过 `&` 符号表示，引用的功能很简单，大部分时候就只是单纯的指向某个值而已。而智能指针则不然，它虽然也号称指针，但是它是一个复杂的家伙：通过比引用更复杂的数据结构，包含比引用更多的信息，例如元数据，当前长度，最大可用长度等。

智能指针和普通引用的区别之一就是所有权的不同。智能指针拥有资源的所有权，而普通引用只是对所有权的借用。

在之前的章节中，实际上我们已经见识过多种智能指针，例如动态字符串 `String` 和动态数组 `Vec`，它们的数据结构中不仅仅包含了指向底层数据的指针，还包含了当前长度、最大长度等信息，其中 `String` 智能指针还提供了一种担保信息：所有的数据都是合法的 `UTF-8` 格式。

智能指针往往是基于结构体实现，它与我们自定义的结构体最大的区别在于它实现了 `Deref` 和 `Drop` 特征：

- `Deref` 可以让智能指针像引用那样工作，这样你就可以写出同时支持智能指针和引用的代码，例如 `*T`
- `Drop` 允许你指定智能指针超出作用域后自动执行的代码，例如做一些数据清除等收尾工作

智能指针在 Rust 中很常见，本章节不是全部，而是挑选几个最常用、最有代表性的进行讲解：

- `Box<T>`，可以将值分配到堆上
- `Rc<T>`，引用计数类型，允许多所有权存在
- `Ref<T>` 和 `RefMut<T>`，允许将借用规则检查从编译期移动到运行期进行

关于裸指针、引用及智能指针的更详细区别将在后续章节中介绍。

## Box<T> 堆对象分配

`Box<T>` 允许你将一个值分配到堆上，然后在栈上保留一个智能指针指向堆上的数据。

之前我们在[所有权章节](https://nange.github.io/posts/2022/04/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E6%89%80%E6%9C%89%E6%9D%83%E5%92%8C%E5%80%9F%E7%94%A8/#%E6%A0%88stack%E4%B8%8E%E5%A0%86heap)简单讲过堆栈的概念，这里再补充一些。

### Rust中的堆栈

栈内存从高位地址向下增长，且栈内存是连续分配的，一般来说**操作系统对栈内存的大小都有限制**，在 Rust 中，`main` 线程的[栈大小是 `8MB`](https://course.rs/pitfalls/stack-overflow.html)，普通线程是 `2MB`，在函数调用时会在其中创建一个临时栈空间，调用结束后 Rust 会让这个栈空间里的对象自动进入 `Drop` 流程，最后栈顶指针自动移动到上一个调用栈顶，无需程序员手动干预，因而栈内存申请和释放是非常高效的。

与栈相反，堆上内存则是从低位地址向上增长，**堆内存通常只受物理内存限制**，而且通常是不连续的，因此从性能的角度看，栈往往比堆更高。

相比其它语言，Rust 堆上对象还有一个特殊之处，它们都拥有一个所有者，因此受所有权规则的限制：当赋值时，发生的是所有权的转移（只需浅拷贝栈上的引用或智能指针即可），例如以下代码：

```rust
fn main() {
    let b = foo("world");
    println!("{}", b);
}

fn foo(x: &str) -> String {
    let a = "Hello, ".to_string() + x;
    a
}
```

在 `foo` 函数中，`a` 是 `String` 类型，它其实是一个智能指针结构体，该智能指针存储在函数栈中，指向堆上的字符串数据。当被从 `foo` 函数转移给 `main` 中的 `b` 变量时，栈上的智能指针被复制一份赋予给 `b`，而底层数据无需发生改变，这样就完成了所有权从 `foo` 函数内部到 `b` 的转移。

#### 堆栈的性能

第一感觉可能会觉得栈的性能肯定比堆高，其实未必。 由于在后面的性能专题会专门讲解堆栈的性能问题，因此这里就大概给出结论：

- 小型数据，在栈上的分配性能和读取性能都要比堆上高
- 中型数据，栈上分配性能高，但是读取性能和堆上并无区别，因为无法利用寄存器或 CPU 高速缓存，最终还是要经过一次内存寻址
- 大型数据，只建议在堆上分配和使用

总之，栈的分配速度肯定比堆上快，但是读取速度往往取决于你的数据能不能放入寄存器或 CPU 高速缓存。 因此不要仅仅因为堆上性能不如栈这个印象，就总是优先选择栈，导致代码更复杂的实现。

### Box的使用场景

由于 `Box` 是简单的封装，除了将值存储在堆上外，并没有其它性能上的损耗。可以在以下场景中使用它：

- 特意的将数据分配在堆上
- 数据较大时，又不想在转移所有权时进行数据拷贝
- 类型的大小在编译期无法确定，但是我们又需要固定大小的类型时
- 特征对象，用于说明对象实现了一个特征，而不是某个特定的类型

#### 使用Box<T>将数据存储在堆上

```rust
fn main() {
    let a = Box::new(3);
    println!("a = {}", a); // a = 3

    // 下面一行代码将报错
    // let b = a + 1; // cannot add `{integer}` to `Box<{integer}>`
}
```

这样就可以创建一个智能指针指向了存储在堆上的 `3`，并且 `a` 持有了该指针。在本章的引言中，我们提到了智能指针往往都实现了 `Deref` 和 `Drop` 特征，因此：

- `println!` 可以正常打印出 `a` 的值，是因为它隐式地调用了 `Deref` 对智能指针 `a` 进行了解引用
- 最后一行代码 `let b = a + 1` 报错，是因为在表达式中，我们无法自动隐式地执行 `Deref` 解引用操作，你需要使用 `*` 操作符 `let b = *a + 1`，来显式的进行解引用
- `a` 持有的智能指针将在作用域结束（`main` 函数结束）时，被释放掉，这是因为 `Box<T>` 实现了 `Drop` 特征

以上的例子在实际代码中其实很少会存在，因为将一个简单的值分配到堆上并没有太大的意义。将其分配在栈上，由于寄存器、CPU 缓存的原因，它的性能将更好，而且代码可读性也更好。

#### 避免栈上数据的拷贝

如果数据是存储在栈上（如i32类型），则转移所有权时，实际上是把数据拷贝了一份，最终新旧变量各自拥有不同的数据，因此所有权并未转移。

而堆上则不然，底层数据并不会被拷贝，转移所有权仅仅是复制一份栈中的指针，再将新的指针赋予新的变量，然后让拥有旧指针的变量失效，最终完成了所有权的转移：

```rust
fn main() {
    // 在栈上创建一个长度为1000的数组
    let arr = [0;1000];
    // 将arr所有权转移arr1，由于 `arr` 分配在栈上，因此这里实际上是直接重新深拷贝了一份数据
    let arr1 = arr;

    // arr 和 arr1 都拥有各自的栈上数组，因此不会报错
    println!("{:?}", arr.len());
    println!("{:?}", arr1.len());

    // 在堆上创建一个长度为1000的数组，然后使用一个智能指针指向它
    let arr = Box::new([0;1000]);
    // 将堆上数组的所有权转移给 arr1，由于数据在堆上，因此仅仅拷贝了智能指针的结构体，底层数据并没有被拷贝
    // 所有权顺利转移给 arr1，arr 不再拥有所有权
    let arr1 = arr;
    println!("{:?}", arr1.len());
    // 由于 arr 不再拥有底层数组的所有权，因此下面代码将报错
    // println!("{:?}", arr.len());
}
```

从以上代码，可以清晰看出大块的数据为何应该放入堆中，此时 `Box` 就成为了我们最好的帮手。

#### 将动态大小类型变为Sized固定大小类型

Rust 需要在编译时知道类型占用多少空间，如果一种类型在编译时无法知道具体的大小，那么被称为动态大小类型 DST。

其中一种无法在编译时知道大小的类型是**递归类型**：在类型定义中又使用到了自身，或者说该类型的值的一部分可以是相同类型的其它值，这种值的嵌套理论上可以无限进行下去，所以 Rust 不知道递归类型需要多少空间：

```rust
enum List {
    Cons(i32, List),
    Nil,
}
```

这种嵌套可以无限进行下去，Rust 认为该类型是一个 DST 类型，并给予报错：

```rust
error[E0072]: recursive type `List` has infinite size //递归类型 `List` 拥有无限长的大小
 --> src/main.rs:3:1
  |
3 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
4 |     Cons(i32, List),
  |               ---- recursive without indirection
```

使用`Box<T>`解决此问题：

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

只需要将 `List` 存储到堆上，然后使用一个智能指针指向它，即可完成从 DST 到 Sized 类型(固定大小类型)的转变。

#### 特征对象

在 Rust 中，想实现不同类型组成的数组只有两个办法：枚举和特征对象，前者限制较多，因此后者往往是最常用的解决办法。

```rust
trait Draw {
    fn draw(&self);
}

struct Button {
    id: u32,
}
impl Draw for Button {
    fn draw(&self) {
        println!("这是屏幕上第{}号按钮", self.id)
    }
}

struct Select {
    id: u32,
}

impl Draw for Select {
    fn draw(&self) {
        println!("这个选择框贼难用{}", self.id)
    }
}

fn main() {
    let elems: Vec<Box<dyn Draw>> = vec![Box::new(Button { id: 1 }), Box::new(Select { id: 2 })];

    for e in elems {
        e.draw()
    }
}
```

以上代码将不同类型的 `Button` 和 `Select` 包装成 `Draw` 特征的特征对象，放入一个数组中，`Box<dyn Draw>` 就是特征对象。

其实，特征也是 DST 类型，而特征对象在做的就是将 DST 类型转换为固定大小类型。

### Box 内存布局

先来看看 `Vec<i32>` 的内存布局：

```rust
(stack)    (heap)
┌──────┐   ┌───┐
│ vec1 │──→│ 1 │
└──────┘   ├───┤
           │ 2 │
           ├───┤
           │ 3 │
           ├───┤
           │ 4 │
           └───┘
```

之前提到过 `Vec` 和 `String` 都是智能指针，从上图可以看出，该智能指针存储在栈中，然后指向堆上的数组数据。

那如果数组中每个元素都是一个 `Box` 对象呢？来看看 `Vec<Box<i32>>` 的内存布局：

```rust
                    (heap)
(stack)    (heap)   ┌───┐
┌──────┐   ┌───┐ ┌─→│ 1 │
│ vec2 │──→│B1 │─┘  └───┘
└──────┘   ├───┤    ┌───┐
           │B2 │───→│ 2 │
           ├───┤    └───┘
           │B3 │─┐  ┌───┐
           ├───┤ └─→│ 3 │
           │B4 │─┐  └───┘
           └───┘ │  ┌───┐
                 └─→│ 4 │
                    └───┘
```

上面的 `B1` 代表被 `Box` 分配到堆上的值 `1`。

可以看出智能指针 `vec2` 依然是存储在栈上，然后指针指向一个堆上的数组，该数组中每个元素都是一个 `Box` 智能指针，最终 `Box` 智能指针又指向了存储在堆上的实际值。

因此当我们从数组中取出某个元素时，取到的是对应的智能指针 `Box`，需要对该智能指针进行解引用，才能取出最终的值：

```rust
fn main() {
    let arr = vec![Box::new(1), Box::new(2)];
    let (first, second) = (&arr[0], &arr[1]);
    let sum = **first + **second;
}
```

以上代码有几个值得注意的点：

- 使用 `&` 借用数组中的元素，否则会报所有权错误（不能从数组中移出item的所有权）
- 表达式不能隐式的解引用，因此必须使用 `**` 做两次解引用，第一次将 `&Box<i32>` 类型转成 `Box<i32>`，第二次将 `Box<i32>` 转成 `i32`

### Box::leak

`Box` 中还提供了一个非常有用的关联函数：`Box::leak`，它可以消费掉 `Box` 并且强制目标值从内存中泄漏，这有啥用啊？

例如，可以把一个 `String` 类型，变成一个 `'static` 生命周期的 `&str` 类型：

```rust
fn main() {
   let s = gen_static_str();
   println!("{}", s);
}

fn gen_static_str() -> &'static str{
    let mut s = String::new();
    s.push_str("hello, world");

    Box::leak(s.into_boxed_str())
}
```

在之前的代码中，如果 `String` 创建于函数中，那么返回它的唯一方法就是转移所有权给调用者 `fn move_str() -> String`，而通过 `Box::leak` 我们不仅返回了一个 `&str` 字符串切片，它还是 `'static` 生命周期的！

一般来说具有 `'static` 生命周期的往往都是编译期就创建的值，例如 `let v = "hello, world"`，这里 `v` 是直接打包到二进制可执行文件中的，因此该字符串具有 `'static` 生命周期，再比如 `const` 常量。

如果手动为变量标注 `'static` 呢？。其实标注的 `'static` 只是用来忽悠编译器的，但是超出作用域，一样被释放回收。而使用 `Box::leak` 就可以将一个运行期的值转为 `'static`。

#### 使用场景

一个简单的场景：**需要一个在运行期初始化的值，但是可以全局有效，也就是和整个程序活得一样久**，那么就可以使用 `Box::leak`，例如有一个存储配置的结构体实例，它是在运行期动态插入内容，那么就可以将其转为全局有效，虽然 `Rc/Arc` 也可以实现此功能，但是 `Box::leak` 是性能最高的。



## Deref解引用

### 通过 * 获取引用背后的值

先看一下常规引用的解引用。

```rust
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(5, x);  
    assert_eq!(5, *y);
    // assert_eq!(5, y);  // 此行报错，不能把引用和数字比较
}
```

这里 `y` 就是一个常规引用，包含了值 `5` 所在的内存地址，然后通过解引用 `*y`，我们获取到了值 `5`。

### 智能指针解引用

 Rust 将解引用提升到了一个新高度。智能指针本质上是一个结构体类型，如果直接对它进行 `*myStruct`，显然编译器不知道该如何办。

实现 `Deref` 后的智能指针结构体，就可以像普通引用一样，通过 `*` 进行解引用，例如 `Box<T>` 智能指针：

```rust
fn main() {
    let x = Box::new(1);
    let sum = *x + 1;
}
```

智能指针 `x` 被 `*` 解引用为 `i32` 类型的值 `1`，然后再进行求和。

#### 定义自己的智能指针

实现一个简化版的`Box<T>`智能指针。

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let y = MyBox::new(5);

    assert_eq!(5, *y);
}
```

### * 背后的原理

当我们对智能指针 `Box` 进行解引用时，实际上 Rust 为我们调用了以下方法：

```rust
*(y.deref())
```

首先调用 `deref` 方法返回值的常规引用，然后通过 `*` 对常规引用进行解引用，最终获取到目标值。

### 函数和方法中的隐式Deref转换

若一个类型实现了 `Deref` 特征，那它的引用在传给函数或方法时，会根据参数签名来决定是否进行隐式的 `Deref` 转换，例如：

```rust
fn main() {
    let s = String::from("hello world");
    display(&s)
}

fn display(s: &str) {
    println!("{}",s);
}
```

以上代码有几点值得注意：

- `String` 实现了 `Deref` 特征，可以在需要时自动被转换为 `&str` 类型
- `&s` 是一个 `&String` 类型，当它被传给 `display` 函数时，自动通过 `Deref` 转换成了 `&str`
- 必须使用 `&s` 的方式来触发 `Deref`(仅引用类型的实参才会触发自动解引用，因为Deref 函数接受的引用类型)

#### 连续的隐式Deref转换

`Deref` 可以支持连续的隐式转换，直到找到适合的形式为止：

```rust
fn main() {
    let s = MyBox::new(String::from("hello world"));
    display(&s)
}

fn display(s: &str) {
    println!("{}",s);
}
```

这种方式提供了代码编写上的简洁性。但也有其缺点：如果不知道某个类型是否实现了 `Deref` 特征，那么在看到某段代码时，并不能在第一时间反应过来该代码发生了隐式的 `Deref` 转换。事实上，不仅仅是 `Deref`，在 Rust 中还有各种 `From/Into` 等等会给阅读代码带来一定负担的特征。一切选择都是权衡，有得必有失，得了代码的简洁性，往往就失去了可读性，Go 语言就是一个刚好相反的例子。

再来看一下在方法、赋值中自动应用 `Deref` 的例子：

```rust
fn main() {
    let s = MyBox::new(String::from("hello, world"));
    let s1: &str = &s;
    let s2: String = s.to_string();
}
```



## Drop释放资源

### Rust的资源回收

在 Rust 中，可以指定在一个变量超出作用域时，执行一段特定的代码，最终编译器将自动插入这段收尾代码。这样，就无需在每一个使用该变量的地方，都写一段代码来进行收尾工作和资源释放。

指定这段收尾工作代码就是靠`Drop`特征。同时它也是智能指针的必备特征之一。

```rust
struct HasDrop1;
struct HasDrop2;
impl Drop for HasDrop1 {
    fn drop(&mut self) {
        println!("Dropping HasDrop1!");
    }
}
impl Drop for HasDrop2 {
    fn drop(&mut self) {
        println!("Dropping HasDrop2!");
    }
}
struct HasTwoDrops {
    one: HasDrop1,
    two: HasDrop2,
}
impl Drop for HasTwoDrops {
    fn drop(&mut self) {
        println!("Dropping HasTwoDrops!");
    }
}

struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!")
    }
}

fn main() {
    let _x = HasTwoDrops {
        two: HasDrop2,
        one: HasDrop1,
    };
    let _foo = Foo;
    println!("Running!");
}
```

输出：

```rust
Running!
Dropping Foo!
Dropping HasTwoDrops!
Dropping HasDrop1!
Dropping HasDrop2!
```

有几点值得注意：

- `Drop` 特征中的 `drop` 方法借用了目标的可变引用，而不是拿走了所有权
- 结构体中每个字段都有自己的 `Drop`

#### Drop的顺序

- **变量级别，按照逆序的方式**，`_x` 在 `_foo` 之前创建，因此 `_x` 在 `_foo` 之后被 `drop`
- **结构体内部，按照顺序的方式**，结构体 `_x` 中的字段按照定义中的顺序依次 `drop`

#### 没有实现Drop的结构体

实际上就算不为 `_x` 结构体实现 `Drop` 特征，它内部的两个字段依然会调用 `drop`。

原因在于，Rust 自动为几乎所有类型都实现了 `Drop` 特征，因此就算不手动为结构体实现 `Drop`，它依然会调用默认实现的 `drop` 函数，同时再调用每个字段的 `drop` 方法。

### Drop使用场景

- 回收内存资源
- 行一些收尾工作

对于第二点，上面已经详细介绍过，因此这里主要对第一点进行下简单说明。

在绝大多数情况下，我们都无需手动去 `drop` 以回收内存资源，因为 Rust 会自动帮我们完成这些工作，它甚至会对复杂类型的每个字段都单独的调用 `drop` 进行回收！但是确实有极少数情况，需要你自己来回收资源的，例如文件描述符、网络 socket 等，当这些值超出作用域不再使用时，就需要进行关闭以释放相关的资源，在这些情况下，就需要使用者自己来解决 `Drop` 的问题。

### 互斥的Copy和Drop

我们无法为一个类型同时实现 `Copy` 和 `Drop` 特征。因为实现了 `Copy` 的特征会被编译器隐式的复制，因此非常难以预测析构函数执行的时间和频率。因此这些实现了 `Copy` 的类型无法拥有析构函数。

```rust
#[derive(Copy)]
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!")
    }
}
```

以上代码报错如下：

```rust
error[E0184]: the trait `Copy` may not be implemented for this type; the type has a destructor
  --> src/main.rs:24:10
   |
24 | #[derive(Copy)]
   |          ^^^^ Copy not allowed on types with destructors
```



### 总结

`Drop` 可以用于许多方面，来使得资源清理及收尾工作变得方便和安全！通过 `Drop` 特征和 Rust 所有权系统，你无需担心之后的代码清理，Rust 会自动考虑这些问题。

我们也无需担心意外的清理掉仍在使用的值，这会造成编译器错误：所有权系统确保引用总是有效的，也会确保 `drop` 只会在值不再被使用时被调用一次。



## Rc与Arc

Rust 所有权机制要求一个值只能有一个所有者，在大多数情况下，都没有问题，但是考虑以下情况：

- 在图数据结构中，多个边可能会拥有同一个节点，该节点直到没有边指向它时，才应该被释放清理
- 在多线程中，多个线程可能会持有同一个数据，但是你受限于 Rust 的安全机制，无法同时获取该数据的可变引用

为了解决此类问题，Rust 在所有权机制之外又引入了额外机制：通过引用计数的方式，允许一个数据资源在同一时刻拥有多个所有者。

这种实现机制就是 `Rc` 和 `Arc`，前者适用于单线程，后者适用于多线程。由于二者大部分情况下都相同，因此本章将以 `Rc` 作为讲解主体，对于 `Arc` 的不同之处，另外进行单独讲解。

### Rc<T>

引用计数(reference counting)，顾名思义，通过记录一个数据被引用的次数来确定该数据是否正在被使用。当引用次数归零时，就代表该数据不再被使用，因此可以被清理释放。

当我们**希望在堆上分配一个对象供程序的多个部分使用且无法确定哪个部分最后一个结束时，就可以使用 `Rc` 成为数据值的所有者**。

下面是经典的所有权被转移导致报错的例子：

```rust
fn main() {
    let s = String::from("hello, world");
    // s在这里被转移给a
    let a = Box::new(s);
    // 报错！此处继续尝试将 s 转移给 b
    let b = Box::new(s);
}
```

使用 `Rc` 就可以轻易解决：

```rust
use std::rc::Rc;
fn main() {
    let a = Rc::new(String::from("hello, world"));
    let b = Rc::clone(&a);

    assert_eq!(2, Rc::strong_count(&a));
    assert_eq!(Rc::strong_count(&a), Rc::strong_count(&b))
}
```

#### Rc::clone

使用 `Rc::clone` 克隆了一份智能指针 `Rc<String>`，并将该智能指针的引用计数增加到 `2`。由于 `a` 和 `b` 是同一个智能指针的两个副本，因此通过它们两个获取引用计数的结果都是 `2`。

这里的 `clone` **仅仅复制了智能指针并增加了引用计数，并没有克隆底层数据**，因此 `a` 和 `b` 是共享了底层的字符串 `s`，这种**复制效率是非常高**的。也可以使用 `a.clone()` 的方式来克隆，但是从可读性角度，更加推荐 `Rc::clone` 的方式。

#### 观察引用计数的变化

使用关联函数 `Rc::strong_count` 可以获取当前引用计数的值。

```rust
use std::rc::Rc;
fn main() {
        let a = Rc::new(String::from("test ref counting"));
        println!("count after creating a = {}", Rc::strong_count(&a));
        let b =  Rc::clone(&a);
        println!("count after creating b = {}", Rc::strong_count(&a));
        {
            let c =  Rc::clone(&a);
            println!("count after creating c = {}", Rc::strong_count(&c));
        }
        println!("count after c goes out of scope = {}", Rc::strong_count(&a));
}
```

有几点值得注意：

- 由于变量 `c` 在语句块内部声明，当离开语句块时它会因为超出作用域而被释放，所以引用计数会减少 1，事实上这个得益于 `Rc<T>` 实现了 `Drop` 特征
- `a`、`b`、`c` 三个智能指针引用计数都是同样的，并且共享底层的数据，因此打印计数时用哪个都行
- 无法看到的是：当 `a`、`b` 超出作用域后，引用计数会变成 0，最终智能指针和它指向的底层字符串都会被清理释放

#### 不可变引用

事实上，`Rc<T>` 是指向底层数据的不可变的引用，因此你无法通过它来修改数据，这也符合 Rust 的借用规则：要么存在多个不可变借用，要么只能存在一个可变借用。

但是实际开发中我们往往需要对数据进行修改，则需要配合其它数据类型来一起使用，例如内部可变性的 `RefCell<T>` 类型以及互斥锁 `Mutex<T>`。事实上，在多线程编程中，`Arc` 跟 `Mutext` 锁的组合使用非常常见，它们既可以让我们在不同的线程中共享数据，又允许在各个线程中对其进行修改。

#### 一个综合例子

考虑一个场景，有很多小工具，每个工具都有自己的主人，但是存在多个工具属于同一个主人的情况，此时使用 `Rc<T>` 就非常适合：

```rust
use std::rc::Rc;

struct Owner {
    name: String,
    // ...其它字段
}

struct Gadget {
    id: i32,
    owner: Rc<Owner>,
    // ...其它字段
}

fn main() {
    // 创建一个基于引用计数的 `Owner`.
    let gadget_owner: Rc<Owner> = Rc::new(Owner {
        name: "Gadget Man".to_string(),
    });

    // 创建两个不同的工具，它们属于同一个主人
    let gadget1 = Gadget {
        id: 1,
        owner: Rc::clone(&gadget_owner),
    };
    let gadget2 = Gadget {
        id: 2,
        owner: Rc::clone(&gadget_owner),
    };

    // 释放掉第一个 `Rc<Owner>`
    drop(gadget_owner);

    // 尽管在上面我们释放了 gadget_owner，但是依然可以在这里使用 owner 的信息
    // 原因是在 drop 之前，存在三个指向 Gadget Man 的智能指针引用，上面仅仅
    // drop 掉其中一个智能指针引用，而不是 drop 掉 owner 数据，外面还有两个
    // 引用指向底层的 owner 数据，引用计数尚未清零
    // 因此 owner 数据依然可以被使用
    println!("Gadget {} owned by {}", gadget1.id, gadget1.owner.name);
    println!("Gadget {} owned by {}", gadget2.id, gadget2.owner.name);

    // 在函数最后，`gadget1` 和 `gadget2` 也被释放，最终引用计数归零，随后底层
    // 数据也被清理释放
}
```

以上代码很好的展示了 `Rc<T>` 的用途，当然也可以用借用的方式，但是实现起来就会复杂得多，而且随着 `Gadget` 在代码的各个地方使用，引用生命周期也将变得更加复杂，毕竟结构体中的引用类型，总是令人不那么愉快。

#### Rc简单总结

- `Rc/Arc` 是不可变引用，你无法修改它指向的值，只能进行读取，如果要修改，需要配合后面章节的内部可变性 `RefCell` 或互斥锁 `Mutex`
- 一旦最后一个拥有者消失，则资源会自动被回收，这个生命周期是在编译期就确定下来的
- `Rc` 只能用于同一线程内部，想要用于线程之间的对象共享，需要使用 `Arc`
- `Rc<T>` 是一个智能指针，实现了 `Deref` 特征，因此你无需先解开 `Rc` 指针，再使用里面的 `T`，而是可以直接使用 `T`，例如上例中的 `gadget1.owner.name`

### 多线程无力的Rc<T>

在多线程场景使用 `Rc<T>` 会如何?：

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let s = Rc::new(String::from("多线程漫游者"));
    for _ in 0..10 {
        let s = Rc::clone(&s);
        let handle = thread::spawn(move || {
           println!("{}", s)
        });
    }
}
```

以上代码会报错：

```rust
error[E0277]: `Rc<String>` cannot be sent between threads safely
```

表面原因是 `Rc<T>` 不能在线程间安全的传递，实际上是因为它没有实现 `Send` 特征，而该特征是恰恰是多线程间传递数据的关键，我们会在多线程章节中进行说明。

当然，还有更深层的原因：由于 `Rc<T>` 需要管理引用计数，但是该计数器并没有使用任何并发原语，因此无法实现原子化的计数操作，最终会导致计数错误。

### Arc

`Arc` 是 `Atomic Rc` 的缩写，顾名思义：原子化的 `Rc<T>` 智能指针。它能保证我们的数据能够安全的在线程间共享。

#### Arc的性能损耗

Arc这么好，为何不直接使用 `Arc`？

原因在于原子化或者其它锁虽然可以带来的线程安全，但是都会伴随着性能损耗，而且这种性能损耗还不小。因此Rust将选择器交给用户。

`Arc` 和 `Rc` 拥有完全一样的 API，修改起来很简单：

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let s = Arc::new(String::from("多线程漫游者"));
    for _ in 0..10 {
        let s = Arc::clone(&s);
        let handle = thread::spawn(move || {
           println!("{}", s)
        });
    }
}
```

两者还有一点区别：`Arc` 和 `Rc` 并没有定义在同一个模块，前者通过 `use std::sync::Arc` 来引入，后者通过 `use std::rc::Rc`。

### 总结

在 Rust 中，所有权机制保证了一个数据只会有一个所有者，但如果你想要在图数据结构、多线程等场景中共享数据，这种机制会成为极大的阻碍。好在 Rust 为我们提供了智能指针 `Rc` 和 `Arc`，使用它们就能实现多个所有者共享一个数据的功能。

`Rc` 和 `Arc` 的区别在于，后者是原子化实现的引用计数，因此是线程安全的，可以用于多线程中共享数据。

这两者都是只读的，如果想要实现内部数据可修改，必须配合内部可变性 `RefCell` 或者互斥锁 `Mutex` 来一起使用。



## Cell和RefCell

TODO：













