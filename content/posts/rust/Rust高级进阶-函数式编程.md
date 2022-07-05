---
title: "[Rust course]-16 高级进阶-函数式编程：闭包、迭代器"
date: 2022-05-23T10:47:00+08:00
lastmod: 2022-05-26T17:02:00+08:00
author: nange
draft: false
description: "Rust函数式编程"

categories: ["编程语言"]
series: ["rust-course"]
series_weight: 16
tags: ["rust"]
---

## 闭包 Closure

闭包是**一种匿名函数，它可以赋值给变量也可以作为参数传递给其它函数，不同于函数的是，它允许捕获调用者作用域中的值**，例如：

```rust
fn main() {
   let x = 1;
   let sum = |y| x + y;

    assert_eq!(3, sum(2));
}
```

上面的代码展示了非常简单的闭包 `sum`，它拥有一个入参 `y`，同时捕获了作用域中的 `x` 的值，因此调用 `sum(2)` 意味着将 2（参数 `y`）跟 1（`x`）进行相加,最终返回它们的和：`3`。

可以看到 `sum` 非常符合闭包的定义：可以赋值给变量，允许捕获调用者作用域中的值。

### 闭包的定义

```rust
|param1, param2,...| {
    语句1;
    语句2;
    返回表达式
}
```

如果只有返回表达式，定义可以简化为：

```rust
|param1| 返回表达式
```

注意点：

* **闭包中最后一行表达式返回的值，就是闭包执行后的返回值**
* `let action = ||...` 只是把闭包赋值给变量 `action`，并不是把闭包执行后的结果赋值给 `action`，因此 `action` 就相当于闭包函数，可以跟函数一样进行调用：`action()`

### 闭包的类型推导

Rust 是静态语言，所有的变量都具有类型，但是得益于编译器的强大类型推导能力，在很多时候我们并不需要显式地去声明类型，但是显然函数并不在此列，必须手动为函数的所有参数和返回值指定类型，原因在于函数往往会作为 API 提供给你的用户，因此你的用户必须在使用时知道传入参数的类型和返回值类型。

与函数相反，闭包并不会作为 API 对外提供，因此它可以享受编译器的类型推导能力，无需标注参数和返回值的类型。

为了增加代码可读性，有时候我们会显式地给类型进行标注，出于同样的目的，也可以给闭包标注类型：

```rust
let sum = |x: i32, y: i32| -> i32 {
    x + y
}
```

与之相比，不标注类型的闭包声明会更简洁些：`let sum = |x, y| x + y`，需要注意的是，针对 `sum` 闭包，如果只进行了声明，但是没有使用，编译器会提示你为 `x, y` 添加类型标注，因为它缺乏必要的上下文：

```rust
let sum  = |x, y| x + y;
let v = sum(1, 2);
```

像这样编译器才可以推导出 `x,y` 的类型为 `i32`。

### 结构体中的闭包

假设我们要实现一个简易缓存，功能是获取一个值，然后将其缓存起来，那么可以这样设计：

* 一个闭包用于获取值
* 一个变量，用于存储该值

可以使用结构体来代表缓存对象，最终设计如下：

```rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    query: T,
    value: Option<u32>,
}
```

等等，`Fn(u32) -> u32` 是什么鬼？好像没见过？

可以看得出这一长串是 `T` 的特征约束，再结合之前的已知信息：`query` 是一个闭包，可以推测出，`Fn(u32) -> u32` 是一个特征，用来表示 `T` 是一个闭包类型。

那为什么不用具体的类型来标注 `query` 呢？原因很简单，每一个闭包实例都有独属于自己的类型，即使于两个签名一模一样的闭包，它们的类型也是不同的，因此无法用一个统一的类型来标注 `query` 闭包。

而标准库提供的 `Fn` 系列特征，再结合特征约束，就能很好的解决了这个问题. `T: Fn(u32) -> u32` 意味着 `query` 的类型是 `T`，该类型必须实现了相应的闭包特征 `Fn(u32) -> u32`。从特征的角度来看它长得非常反直觉，但是如果从闭包的角度来看又极其符合直觉，不得不佩服 Rust 团队的鬼才设计。。。

特征 `Fn(u32) -> u32` 从表面来看，就对闭包形式进行了显而易见的限制：**该闭包拥有一个`u32`类型的参数，同时返回一个`u32`类型的值**。

> 需要注意的是，其实 Fn 特征不仅仅适用于闭包，还适用于函数，因此上面的 `query` 字段除了使用闭包作为值外，还能使用一个具名的函数来作为它的值

接着，为缓存实现方法：

```rust
impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(query: T) -> Cacher<T> {
        Cacher {
            query,
            value: None,
        }
    }

    // 先查询缓存值 `self.value`，若不存在，则调用 `query` 加载
    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.query)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```

### 捕获作用域中的值

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;
    // fn equal_to_x(z: i32) -> bool {  
    //     z == x // 使用函数捕获就会报错
    // }

    let y = 4;

    assert!(equal_to_x(y));
}
```

上面代码中，`x` 并不是闭包 `equal_to_x` 的参数，但是它依然可以去使用 `x`，因为 `equal_to_x` 在 `x` 的作用域范围内。

#### 闭包对内存的影响

当闭包从环境中捕获一个值时，会分配内存去存储这些值。对于有些场景来说，这种额外的内存分配会成为一种负担。与之相比，函数就不会去捕获这些环境值，因此定义和使用函数不会拥有这种内存负担。

#### 三种Fn特征

闭包捕获变量有三种途径，恰好对应函数参数的三种传入方式：转移所有权、可变借用、不可变借用，因此相应的 `Fn` 特征也有三种：

##### FnOnce

`Once` 顾名思义，说明该闭包只能运行一次：

```rust
fn fn_once<F>(func: F)
where
    F: FnOnce(usize) -> bool,
{
    println!("{}", func(3));
    println!("{}", func(4));
}

fn main() {
    let x = vec![1, 2, 3];
    fn_once(|z|{z == x.len()})
}
```

**仅**实现 `FnOnce` 特征的闭包在调用时会转移所有权，所以显然不能对已失去所有权的闭包变量进行二次调用：

```rust
error[E0382]: use of moved value: `func`
 --> src\main.rs:6:20
  |
1 | fn fn_once<F>(func: F)
  |               ---- move occurs because `func` has type `F`, which does not implement the `Copy` trait
                  // 因为`func`的类型是没有实现`Copy`特性的 `F`，所以发生了所有权的转移
...
5 |     println!("{}", func(3));
  |                    ------- `func` moved due to this call // 转移在这
6 |     println!("{}", func(4));
  |                    ^^^^ value used here after move // 转移后再次用
  |
```

这里面有一个很重要的提示，因为 `F` 没有实现 `Copy` 特征，所以会报错，那么我们添加一个约束，试试实现了 `Copy` 的闭包：

```rust
fn fn_once<F>(func: F)
where
    F: FnOnce(usize) -> bool + Copy,// 改动在这里
{
    println!("{}", func(3));
    println!("{}", func(4));
}

fn main() {
    let x = vec![1, 2, 3];
    fn_once(|z|{z == x.len()})
}
```

上面代码中，`func` 的类型 `F` 实现了 `Copy` 特征，调用时使用的将是它的拷贝，所以并没有发生所有权的转移。

```rust
true
false
```

如果你想强制闭包取得捕获变量的所有权，可以在参数列表前添加 `move` 关键字，这种用法通常用于闭包的生命周期大于捕获变量的生命周期时，例如将闭包返回或移入其他线程。

```rust
use std::thread;
let v = vec![1, 2, 3];
let handle = thread::spawn(move || {
    println!("Here's a vector: {:?}", v);
});
handle.join().unwrap();
```

##### FnMut

以可变借用方式修改捕获的变量时，闭包则默认实现`FnMut`。因此可以修改该值：

```rust
fn main() {
    let mut s = String::new();

    let update_string =  |str| s.push_str(str);
    update_string("hello");

    println!("{:?}",s);
}
```

在闭包中，我们调用 `s.push_str` 去改变外部 `s` 的字符串值，因此这里捕获了它的可变借用，运行下试试：

```rust
error[E0596]: cannot borrow `update_string` as mutable, as it is not declared as mutable
 --> src/main.rs:5:5
  |
4 |     let update_string =  |str| s.push_str(str);
  |         -------------          - calling `update_string` requires mutable binding due to mutable borrow of `s`
  |         |
  |         help: consider changing this to be mutable: `mut update_string`
5 |     update_string("hello");
  |     ^^^^^^^^^^^^^ cannot borrow as mutable
```

虽然报错了，但是编译器给出了非常清晰的提示，想要在闭包内部捕获可变借用，需要把该闭包声明为可变类型，也就是 `update_string` 要修改为 `mut update_string`：

```rust
fn main() {
    let mut s = String::new();

    let mut update_string =  |str| s.push_str(str);
    update_string("hello");

    println!("{:?}",s);
}
```

这种写法有点反直觉，相比起来前面的 `move` 更符合使用和阅读习惯。但是如果你忽略 `update_string` 的类型，仅仅把它当成一个普通变量，那么这种声明就比较合理了。

再来看一个复杂点的：

```rust
fn main() {
    let mut s = String::new();

    let update_string =  |str| s.push_str(str);

    exec(update_string);

    println!("{:?}",s);
}

fn exec<'a, F: FnMut(&'a str)>(mut f: F)  {
    f("hello")
}
```

这段代码非常清晰的说明了 `update_string` 实现了 `FnMut` 特征。

##### Fn

以不可变借用的方式捕获环境中的值时，闭包默认实现`Fn`。 让我们把上面的代码中 `exec` 的 `F` 泛型参数类型修改为 `Fn(&'a str)`：

```rust
fn main() {
    let mut s = String::new();

    let update_string =  |str| s.push_str(str);

    exec(update_string);

    println!("{:?}",s);
}

fn exec<'a, F: Fn(&'a str)>(mut f: F)  {
    f("hello")
}
```

运行看看结果：

```rust
error[E0525]: expected a closure that implements the `Fn` trait, but this closure only implements `FnMut`
 --> src/main.rs:4:26  // 期望闭包实现的是`Fn`特征，但是它只实现了`FnMut`特征
  |
4 |     let update_string =  |str| s.push_str(str);
  |                          ^^^^^^-^^^^^^^^^^^^^^
  |                          |     |
  |                          |     closure is `FnMut` because it mutates the variable `s` here
  |                          this closure implements `FnMut`, not `Fn` //闭包实现的是FnMut，而不是Fn
5 |
6 |     exec(update_string);
  |     ---- the requirement to implement `Fn` derives from here
```

从报错中很清晰的看出，我们的闭包实现的是 `FnMut` 特征，需要的是可变借用，但是在 `exec` 中却给它标注了 `Fn` 特征，因此产生了不匹配，再来看看正确的不可变借用方式：

```rust
fn main() {
    let s = "hello, ".to_string();

    let update_string =  |str| println!("{},{}",s,str);

    exec(update_string);

    println!("{:?}",s);
}

fn exec<'a, F: Fn(String) -> ()>(f: F)  {
    f("world".to_string())
}
```

#### move 和 Fn

实际上使用了 `move` 的闭包依然可能实现了 `Fn` 或 `FnMut` 特征。因为，**一个闭包实现了哪种 Fn 特征只取决于该闭包如何使用被捕获的变量，而不是取决于闭包如何捕获它们**。`move` 本身强调的就是后者，闭包如何捕获变量：

```rust
fn main() {
    let s = String::new();

    let update_string =  move || println!("{}",s);

    exec(update_string);
}

fn exec<F: FnOnce()>(f: F)  {
    f()
}
```

我们在上面的闭包中使用了 `move` 关键字，所以我们的闭包捕获了它，但是由于闭包对 `s` 的使用仅仅是不可变借用，因此该闭包实际上**还**实现了 `Fn` 特征。

上段中使用了一个 `还` 字，这是什么意思呢？因为该闭包不仅仅实现了 `FnOnce` 特征，还实现了 `Fn` 特征，将代码修改成下面这样，依然可以编译：

```rust
fn main() {
    let s = String::new();

    let update_string =  move || println!("{}",s);

    exec(update_string);
}

fn exec<F: Fn()>(f: F)  {
    f()
}
```

从`Fn` trait的源码定义中就可以看出，`FnOnce`是`Fn`的父级trait(supertrait)，因此实现了`Fn`也一定实现了`FnOnce`。

#### 三种Fn的关系

当创建一个闭包时，Rust 根据其如何使用环境中变量来推断闭包该实现哪个trait。由于所有闭包都可以被调用至少一次，因此所有闭包都实现了 `FnOnce` 。那些并没有移动被捕获变量的所有权到闭包内的闭包也实现了 `FnMut` ，而不需要对被捕获的变量进行可变访问的闭包则实现了 `Fn` 。

> 注意：即使使用了`move`捕获变量，闭包也可能实现了`Fn`或者`FnMut`，这是因为闭包实现哪个trait是由闭包怎么使用捕获到的变量决定的，而不是如何捕获变量决定。`move`关键字仅仅代表了后者。

可能有个疑问，为什么所有闭包都要实现`FnOnce`,也就是这样的层级关系：`Fn` -> `FnMut` -> `FnOnce`？让三个trait是平级的关系岂不是更清晰易懂？ 仔细思考就会发现，这三个trait在逻辑上其实就存在包含关系。`FnOnce`代表能且只能调用一次，显然`Fn`,`FnMut`对象，都能满足此要求，实现这个接口很容易，也就是`FnOnce`约束的地方，放上`Fn`,`FnMut`对象，完全可以；其次`FnMut`约束代表，具有能够改变环境变量的能力，也即对应`&mut self`，而实现`Fn`trait的对象完全可以实现`FnMut`，只是内部没有去改变环境变量而已（就像`&mut self`的方法内部，也可以不做修改嘛），因此`FnMut`约束的地方，放上`Fn`对象也完全可以。

一段代码来解释上述内容：

```rust
fn main() {
    let s = String::new();

    let update_string =  || println!("{}",s);

    exec(update_string);
    exec1(update_string);
    exec2(update_string);
}

fn exec<F: FnOnce()>(f: F)  {
    f()
}

fn exec1<F: FnMut()>(mut f: F)  {
    f()
}

fn exec2<F: Fn()>(f: F)  {
    f()
}
```

虽然，闭包只是对 `s` 进行了不可变借用，实际上，它可以适用于任何一种 `Fn` 特征：三个 `exec` 函数说明了一切。

再看一个示例：

```rust
fn main() {
    let mut s = String::new();

    let update_string = |str| -> String {s.push_str(str); s };

    exec(update_string);
}

fn exec<'a, F: FnMut(&'a str) -> String>(mut f: F) {
    f("hello");
}
```

运行：

```rust
5 |     let update_string = |str| -> String {s.push_str(str); s };
  |                         ^^^^^^^^^^^^^^^                   - closure is `FnOnce` because it moves the variable `s` out of its environment
  |                                                           // 闭包实现了`FnOnce`，因为它从捕获环境中移出了变量`s`
  |                         |
  |                         this closure implements `FnOnce`, not `FnMut`
```

此例中，闭包从捕获环境中移出了变量 `s` 的所有权，因此这个闭包仅自动实现了 `FnOnce`，未实现 `FnMut` 和 `Fn`。再次印证之前讲的**一个闭包实现了哪种 Fn 特征取决于该闭包如何使用被捕获的变量，而不是取决于闭包如何捕获它们**，跟是否使用 `move` 没有必然联系。

有个疑问是上面代码不是也对变量`s`做了修改吗？ 为啥没有实现`FnMut`呢？ 原因就在于其先对`s`做了所有权转移，已经是在闭包内部修改`s`了，而不是修改原来环境里面的变量（修改环境变量应该对应`&mut self`，不会发生所有权转移），比如我们修改如下就正常编译：

```rust
fn main() {
    let mut s = String::new();

    let update_string = |str| -> String {
        let s = &mut s;
        s.push_str(str);
        s.to_string()
    };

    exec(update_string);
}

fn exec<'a, F: FnMut(&'a str) -> String>(mut f: F) {
    f("hello");
}
```

可进一步看看三个特征的简化版源码：

```rust
pub trait Fn<Args> : FnMut<Args> {
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

pub trait FnMut<Args> : FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait FnOnce<Args> {
    type Output;

    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
```

从特征约束能看出来 `Fn` 的前提是实现 `FnMut`，`FnMut` 的前提是实现 `FnOnce`，因此要实现 `Fn` 就要同时实现 `FnMut` 和 `FnOnce`，这段源码从侧面印证了之前规则的正确性。从源码中也能看出：`Fn` 获取 `&self`，`FnMut` 获取 `&mut self`，而 `FnOnce` 获取 `self`。

### 闭包作为函数返回值

先看一段代码：

```rust
fn factory() -> Fn(i32) -> i32 {
    let num = 5;

    |x| x + num
}

fn main() {
    let f = factory();

    let answer = f(1);
    assert_eq!(6, answer);
}
```

上面这段代码看起来还是蛮正常的，用 `Fn(i32) -> i32` 特征来代表 `|x| x + num`，非常合理嘛, 可惜理想总是难以照进现实，编译器给我们报了一大堆错误，先挑几个重点来看看：

```rust
fn factory<T>() -> Fn(i32) -> i32 {
  |                    ^^^^^^^^^^^^^^ doesn't have a size known at compile-time // 该类型在编译器没有固定的大小
```

Rust 要求函数的参数和返回类型，必须有固定的内存大小，例如 `i32` 就是 4 个字节，引用类型是 8 个字节，总之，绝大部分类型都有固定的大小，但是不包括特征，因为特征类似接口，对于编译器来说，无法知道它后面藏的真实类型是什么，因为也无法得知具体的大小。

同样，我们也无法知道闭包的具体类型，该怎么办呢？再看看报错提示：

```rust
help: use `impl Fn(i32) -> i32` as the return type, as all return paths are of type `[closure@src/main.rs:11:5: 11:21]`, which implements `Fn(i32) -> i32`
  |
8 | fn factory<T>() -> impl Fn(i32) -> i32 {
```

嗯，编译器提示我们加一个 `impl` 关键字，哦，这样一说，读者可能就想起来了，`impl Trait` 可以用来返回一个实现了指定特征的类型，那么这里 `impl Fn(i32) -> i32` 的返回值形式，说明我们要返回一个闭包类型，它实现了 `Fn(i32) -> i32` 特征。

完美解决，但是，在[特征](https://nange.github.io/posts/2022/05/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E6%B3%9B%E5%9E%8B%E5%92%8C%E7%89%B9%E5%BE%81/#%E5%87%BD%E6%95%B0%E8%BF%94%E5%9B%9E%E4%B8%AD%E7%9A%84-impl-trait)那一章，我们提到过，`impl Trait` 的返回方式有一个非常大的局限，就是你只能返回同样的类型，例如：

```rust
fn factory(x:i32) -> impl Fn(i32) -> i32 {

    let num = 5;

    if x > 1{
        move |x| x + num
    } else {
        move |x| x - num
    }
}
```

编译器报错：

```rust
error[E0308]: `if` and `else` have incompatible types
  --> src/main.rs:15:9
   |
12 | /     if x > 1{
13 | |         move |x| x + num
   | |         ---------------- expected because of this
14 | |     } else {
15 | |         move |x| x - num
   | |         ^^^^^^^^^^^^^^^^ expected closure, found a different closure
16 | |     }
   | |_____- `if` and `else` have incompatible types
   |
```

本章节前面有提到：就算签名一样的闭包，类型也是不同的，因此在这种情况下，就无法再使用 `impl Trait` 的方式去返回闭包。

怎么办？再看看编译器提示，里面有这样一行小字：

```rust
= help: consider boxing your closure and/or using it as a trait object
```

可以用特征对象！只需要用 `Box` 的方式即可实现：

```rust
fn factory(x:i32) -> Box<dyn Fn(i32) -> i32> {
    let num = 5;

    if x > 1 {
        Box::new(move |x| x + num)
    } else {
        Box::new(move |x| x - num)
    }
}
```

至此，闭包作为函数返回值就已完美解决。

## 迭代器 Iterator

迭代器允许我们迭代一个连续的集合，例如数组、动态数组 `Vec`、`HashMap` 等，在此过程中，只需关心集合中的元素如何处理，而无需关心如何开始、如何结束、按照什么样的索引去访问等问题。

### For循环与迭代器

从用途来看，迭代器跟 `for` 循环颇为相似，都是去遍历一个集合，但是实际上它们存在不小的差别，最主要的差别就是：**是否通过索引来访问集合**。

Rust 中的 `for`：

```rust
let arr = [1, 2, 3];
for v in arr {
    println!("{}",v);
}
```

`Rust`中没有使用索引，它把 `arr` 数组当成一个迭代器，直接去遍历其中的元素，从哪里开始，从哪里结束，都无需操心。因此严格来说，Rust 中的 `for` 循环是编译器提供的语法糖，最终还是对迭代器中的元素进行遍历。

在 Rust 中数组是迭代器吗？因为在上面的代码中直接对数组 `arr` 进行了迭代，答案是 `No`。那既然数组不是迭代器，为啥咱可以对它的元素进行迭代呢？原因是数组实现了 `IntoIterator` 特征，Rust 通过 `for` 语法糖，自动把实现了该特征的数组类型转换为迭代器（也可以为自己的集合类型实现此特征），最终让我们可以直接对一个数组进行迭代，类似的还有：

```rust
for i in 1..10 {
    println!("{}", i);
}
```

直接对数值序列进行迭代，也是很常见的使用方式。

`IntoIterator` 特征拥有一个 `into_iter` 方法，因此我们还可以显式的把数组转换成迭代器：

```rust
let arr = [1, 2, 3];
for v in arr.into_iter() {
    println!("{}", v);
}
```

迭代器是函数语言的核心特性，它赋予了 Rust 远超于循环的强大表达能力。

### 惰性初始化

在 Rust 中，迭代器是惰性的，意味着如果你不使用它，那么它将不会发生任何事：

```rust
let v1 = vec![1, 2, 3];

let v1_iter = v1.iter();

for val in v1_iter {
    println!("{}", val);
}
```

在 `for` 循环之前，我们只是简单的创建了一个迭代器 `v1_iter`，此时不会发生任何迭代行为，只有在 `for` 循环开始后，迭代器才会开始迭代其中的元素，最后打印出来。

这种惰性初始化的方式确保了创建迭代器不会有任何额外的性能损耗，其中的元素也不会被消耗，只有使用到该迭代器的时候，一切才开始。

### next方法

`for`是如何取出迭代器中的元素的？

先来看看`Iterator`特征：

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // 省略其余有默认实现的方法
}
```

**迭代器之所以成为迭代器，就是因为实现了 `Iterator` 特征**，要实现该特征，最主要的就是实现其中的 `next` 方法，该方法控制如何从集合中取值，最终返回值的类型是[关联类型](https://nange.github.io/posts/2022/05/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E6%B3%9B%E5%9E%8B%E5%92%8C%E7%89%B9%E5%BE%81/#%E5%85%B3%E8%81%94%E7%B1%BB%E5%9E%8B) `Item`。

因此，之前问题的答案已经很明显：`for` 循环通过不停调用迭代器上的 `next` 方法，来获取迭代器中的元素。

也可以手动调用`next`方法：

```rust
fn main() {
    let arr = [1, 2, 3];
    let mut arr_iter = arr.into_iter();

    assert_eq!(arr_iter.next(), Some(1));
    assert_eq!(arr_iter.next(), Some(2));
    assert_eq!(arr_iter.next(), Some(3));
    assert_eq!(arr_iter.next(), None);
}
```

上面代码有几点需要注意：

* `next` 方法返回的是 `Option` 类型，当有值时返回 `Some(i32)`，无值时返回 `None`
* 遍历是按照迭代器中元素的排列顺序依次进行的，因此我们严格按照数组中元素的顺序取出了 `Some(1)`，`Some(2)`，`Some(3)`
* 手动迭代必须将迭代器声明为 `mut` 可变，因为调用 `next` 会改变迭代器其中的状态数据（当前遍历的位置等），而 `for` 循环去迭代则无需标注 `mut`，因为它会帮我们自动完成

总之，`next` 方法对**迭代器的遍历是消耗性的**，每次消耗其一个元素，最终迭代器中将没有任何元素，只能返回 `None`。

### IntoIterator 特征

#### into_iter, iter, iter_mut

在之前的代码中，我们统一使用了 `into_iter` 的方式将数组转化为迭代器，除此之外，还有 `iter` 和 `iter_mut`，这三者的区别：

* `into_iter` 会夺走所有权
* `iter` 是借用
* `iter_mut` 是可变借用

一般在Rust里面，`into_` 之类的，都是拿走所有权，`_mut` 之类的都是可变借用，剩下的就是不可变借用。

使用一段代码来解释下：

```rust
fn main() {
    let values = vec![1, 2, 3];

    for v in values.into_iter() {
        println!("{}", v)
    }

    // 下面的代码将报错，因为 values 的所有权在上面 `for` 循环中已经被转移走
    // println!("{:?}",values);

    let values = vec![1, 2, 3];
    let _values_iter = values.iter();

    // 不会报错，因为 values_iter 只是借用了 values 中的元素
    println!("{:?}", values);

    let mut values = vec![1, 2, 3];
    // 对 values 中的元素进行可变借用
    let mut values_iter_mut = values.iter_mut();

    // 取出第一个元素，并修改为0
    if let Some(v) = values_iter_mut.next() {
        *v = 0;
    }

    // 输出[0, 2, 3]
    println!("{:?}", values);
}
```

有两点需要注意的是：

* `.iter()` 方法实现的迭代器，调用 `next` 方法返回的类型是 `Some(&T)`
* `.iter_mut()` 方法实现的迭代器，调用 `next` 方法返回的类型是 `Some(&mut T)`，因此在 `if let Some(v) = values_iter_mut.next()` 中，`v` 的类型是 `&mut i32`，最终我们可以通过 `*v = 0` 的方式修改其值

#### Iterator和IntoIterator的区别

这两个其实还蛮容易搞混的，但我们只需要记住，`Iterator` 就是迭代器特征，只有实现了它才能称为迭代器，才能调用 `next`。

而 `IntoIterator` 强调的是某一个类型如果实现了该特征，它可以通过 `into_iter`，`iter` 等方法变成一个迭代器。

### 消费者与适配器

消费者是迭代器上的方法，它会消费掉迭代器中的元素，然后返回其类型的值，这些消费者都有一个共同的特点：在它们的定义中，都依赖 `next` 方法来消费元素。

#### 消费者迭代器

只要迭代器上的某个方法在其内部调用了 `next` 方法，那么此方法就被称为**消费性适配器**：因为 `next` 方法会消耗掉迭代器上的元素，所以方法的调用也会消耗掉迭代器上的元素。

其中一个例子是 `sum` 方法，它会拿走迭代器的所有权，然后通过不断调用 `next` 方法对里面的元素进行求和：

```rust
fn main() {
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    let total: i32 = v1_iter.sum();

    assert_eq!(total, 6);

    // v1_iter 是借用了 v1，因此 v1 可以照常使用
    println!("{:?}",v1);

    // 以下代码会报错，因为 `sum` 拿到了迭代器 `v1_iter` 的所有权
    // println!("{:?}",v1_iter);
}
```

从 `sum` 源码中也可以看出，`self` 类型的方法参数拿走了所有权：

```rust
fn sum<S>(self) -> S
    where
        Self: Sized,
        S: Sum<Self::Item>,
{
    Sum::sum(self)
}
```

#### 迭代器适配器

既然消费者适配器是消费掉迭代器，然后返回一个值。那么迭代器适配器，顾名思义，会返回一个新的迭代器，这是实现链式方法调用的关键：`v.iter().map().filter()...`。

与消费者适配器不同，迭代器适配器是惰性的，意味着你**需要一个消费者适配器来收尾，最终将迭代器转换成一个具体的值**：

```rust
let v1: Vec<i32> = vec![1, 2, 3];

v1.iter().map(|x| x + 1);
```

运行：

```rust
warning: unused `Map` that must be used
 --> src/main.rs:4:5
  |
4 |     v1.iter().map(|x| x + 1);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_must_use)]` on by default
  = note: iterators are lazy and do nothing unless consumed // 迭代器 map 是惰性的，这里不产生任何效果
```

如上述中文注释所说，这里的 `map` 方法是一个迭代器适配器，它是惰性的，不产生任何行为，因此我们还需要一个消费者适配器进行收尾：

```rust
let v1: Vec<i32> = vec![1, 2, 3];

let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

assert_eq!(v2, vec![2, 3, 4]);
```

#### collect

上面代码中，使用了 `collect` 方法，该方法就是一个消费者适配器，使用它可以将一个迭代器中的元素收集到指定类型中，这里我们为 `v2` 标注了 `Vec<_>` 类型，就是为了告诉 `collect`：请把迭代器中的元素消费掉，然后把值收集成 `Vec<_>` 类型，至于为何使用 `_`，因为编译器会帮我们自动推导。

为何 `collect` 在消费时要指定类型？是因为该方法其实很强大，可以收集成多种不同的集合类型，`Vec<T>` 仅仅是其中之一，因此我们必须显式的告诉编译器我们想要收集成的集合类型。

还有一点值得注意，`map` 会对迭代器中的每一个值进行一系列操作，然后把该值转换成另外一个新值，该操作是通过闭包 `|x| x + 1` 来完成：最终迭代器中的每个值都增加了 `1`，从 `[1, 2, 3]` 变为 `[2, 3, 4]`。

再来看看如何使用 `collect` 收集成 `HashMap` 集合：

```rust
use std::collections::HashMap;

fn main() {
    let names = ["sunface", "sunfei"];
    let ages = [18, 18];
    let folks: HashMap<_, _> = names.into_iter().zip(ages.into_iter()).collect();

    println!("{:?}",folks);
}
```

`zip` 是一个迭代器适配器，它的作用就是将两个迭代器的内容压缩到一起，形成 `Iterator<Item=(ValueFromA, ValueFromB)>` 这样的新的迭代器，在此处就是形如 `[(name1, age1), (name2, age2)]` 的迭代器。

然后再通过 `collect` 将新迭代器中`(K, V)` 形式的值收集成 `HashMap<K, V>`，同样的，这里必须显式声明类型，然后 `HashMap` 内部的 `KV` 类型可以交给编译器去推导，最终编译器会推导出 `HashMap<&str, i32>`。

#### 闭包作为适配器参数

之前的 `map` 方法中，我们使用闭包来作为迭代器适配器的参数，它最大的好处不仅在于可以就地实现迭代器中元素的处理，还在于可以捕获环境值：

```rust
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter().filter(|s| s.size == shoe_size).collect()
}
```

`filter` 是迭代器适配器，用于对迭代器中的每个值进行过滤。 它使用闭包作为参数，该闭包的参数 `s` 是来自迭代器中的值，然后使用 `s` 跟外部环境中的 `shoe_size` 进行比较，若相等，则在迭代器中保留 `s` 值，若不相等，则从迭代器中剔除 `s` 值，最终通过 `collect` 收集为 `Vec<Shoe>` 类型。

### 实现Iterator特征

之前的内容我们一直基于数组来创建迭代器，实际上，不仅仅是数组，基于其它集合类型一样可以创建迭代器，例如 `HashMap`。 也可以创建自己的迭代器 —— 只要为自定义类型实现 `Iterator` 特征即可。

首先，创建一个计数器并为其实现迭代器：

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

最后，使用我们新建的 `Counter` 进行迭代：

```rust
let mut counter = Counter::new();

assert_eq!(counter.next(), Some(1));
assert_eq!(counter.next(), Some(2));
assert_eq!(counter.next(), Some(3));
assert_eq!(counter.next(), Some(4));
assert_eq!(counter.next(), Some(5));
assert_eq!(counter.next(), None);
```

#### 实现Iterator特征的其他方法

可以看出，实现自己的迭代器非常简单，但是 `Iterator` 特征中，不仅仅是只有 `next` 一个方法，那为什么我们只需要实现它呢？因为其它方法都具有[默认实现](https://nange.github.io/posts/2022/05/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E6%B3%9B%E5%9E%8B%E5%92%8C%E7%89%B9%E5%BE%81/#%E9%BB%98%E8%AE%A4%E5%AE%9E%E7%8E%B0)，所以无需像 `next` 这样手动去实现，而且这些默认实现的方法其实都是基于 `next` 方法实现的。

下面的代码演示了部分方法的使用：

```rust
let sum: u32 = Counter::new()
    .zip(Counter::new().skip(1))
    .map(|(a, b)| a * b)
    .filter(|x| x % 3 == 0)
    .sum();
assert_eq!(18, sum);
```

其中 `zip`，`map`，`filter` 是迭代器适配器：

* `zip` 把两个迭代器合并成一个迭代器，新迭代器中，每个元素都是一个元组，由之前两个迭代器的元素组成。例如将**形如** `[1, 2, 3]` 和 `[4, 5, 6]` 的迭代器合并后，新的迭代器形如 `[(1, 4),(2, 5),(3, 6)]`
* `map` 是将迭代器中的值经过映射后，转换成新的值
* `filter` 对迭代器中的元素进行过滤，若闭包返回 `true` 则保留元素，反之剔除

而 `sum` 是消费者适配器，对迭代器中的所有元素求和，最终返回一个 `u32` 值 `18`。

### enumerate

在之前的流程控制章节，针对 `for` 循环，我们提供了一种方法可以获取迭代时的索引：

```rust
let v = vec![1u64, 2, 3, 4, 5, 6];
for (i,v) in v.iter().enumerate() {
    println!("第{}个值是{}",i,v)
}
```

第一眼看到此方法是很迷茫的，学习本章节后就明白了，首先 `v.iter()` 创建迭代器，其次 调用 `Iterator` 特征上的方法 `enumerate`，该方法产生一个新的迭代器，其中每个元素均是元组 `(索引，值)`。

因为 `enumerate` 是迭代器适配器，因此我们可以对它返回的迭代器调用其它 `Iterator` 特征方法：

```rust
fn main() {
    let v = vec![1u64, 2, 3, 4, 5, 6];
    let val = v
        .iter()
        .enumerate()
        // 每两个元素剔除一个
        // [1, 3, 5]
        .filter(|&(idx, _)| idx % 2 == 0)
        .map(|(_idx, val)| val)
        // 累加 1+3+5 = 9
        .fold(0u64, |sum, acm| sum + acm);

    println!("{}", val);
}
```

### 迭代器性能

前面提到，要完成集合遍历，既可以使用 `for` 循环也可以使用迭代器，那么二者之间该怎么选择呢，性能有多大差距呢？

直接上测试：

```rust
#![feature(test)]

extern crate rand;
extern crate test;

fn sum_for(x: &[f64]) -> f64 {
    let mut result: f64 = 0.0;
    for i in 0..x.len() {
        result += x[i];
    }
    result
}

fn sum_iter(x: &[f64]) -> f64 {
    x.iter().sum::<f64>()
}

#[cfg(test)]
mod bench {
    use test::Bencher;
    use rand::{Rng,thread_rng};
    use super::*;

    const LEN: usize = 1024*1024;

    fn rand_array(cnt: u32) -> Vec<f64> {
        let mut rng = thread_rng();
        (0..cnt).map(|_| rng.gen::<f64>()).collect()
    }

    #[bench]
    fn bench_for(b: &mut Bencher) {
        let samples = rand_array(LEN as u32);
        b.iter(|| {
            sum_for(&samples)
        })
    }

    #[bench]
    fn bench_iter(b: &mut Bencher) {
        let samples = rand_array(LEN as u32);
        b.iter(|| {
            sum_iter(&samples)
        })
    }
}
```

上面的代码对比了 `for` 循环和迭代器 `iterator` 完成同样的求和任务的性能对比，可以看到迭代器还要更快一点。

```rust
test bench::bench_for  ... bench:     998,331 ns/iter (+/- 36,250)
test bench::bench_iter ... bench:     983,858 ns/iter (+/- 44,673)
```

迭代器是 Rust 的 **零成本抽象**（zero-cost abstractions）之一，意味着抽象并不会引入运行时开销，这与 `Bjarne Stroustrup`（C++ 的设计和实现者）在 `Foundations of C++（2012）` 中所定义的 **零开销**（zero-overhead）如出一辙：

> In general, C++ implementations obey the zero-overhead principle: What you don’t use, you don’t pay for. And further: What you do use, you couldn’t hand code any better.
>
> 一般来说，C++的实现遵循零开销原则：没有使用时，你不必为其买单。 更进一步说，需要使用时，你也无法写出更优的代码了。 （翻译一下：用就完事了）

总之，迭代器是 Rust 受函数式语言启发而提供的高级语言特性，可以写出更加简洁、逻辑清晰的代码。编译器还可以通过循环展开（Unrolling）、向量化、消除边界检查等优化手段，使得迭代器和 `for` 循环都有极为高效的执行效率。

所以请放心大胆的使用迭代器，在获得更高的表达力的同时，也不会导致运行时的损失，何乐而不为呢！

迭代器中还有很多其他方法，后续再继续学习。
