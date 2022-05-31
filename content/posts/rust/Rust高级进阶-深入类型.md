---
title: "[Rust course]-17 高级进阶-深入类型"
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

## newtype 和 类型别名

### newtype

何为 `newtype`？简单来说，就是使用[元组结构体](https://nange.github.io/posts/2022/05/rust%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8-%E5%A4%8D%E5%90%88%E7%B1%BB%E5%9E%8B/#%E5%85%83%E7%BB%84%E7%BB%93%E6%9E%84%E4%BD%93tuple-struct)的方式将已有的类型包裹起来：`struct Meters(u32);`，那么此处 `Meters` 就是一个 `newtype`。

为何需要 `newtype`？

- 自定义类型可以让我们给出更有意义和可读性的类型名，例如与其使用 `u32` 作为距离的单位类型，我们可以使用 `Meters`，它的可读性要好得多
- 对于某些场景，只有 `newtype` 可以很好地解决
- 隐藏内部类型的细节

从第二点说起。

#### 为外部类型实现外部特征

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

#### 更好的可读性及类型异化

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

#### 隐藏内部类型的细节

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

### 类型别名(Type Alias)

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

### ! 永不返回类型

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

## Sized 和 不定长类型DST

在 Rust 中类型有多种抽象的分类方式，例如本书之前章节的：基本类型、集合类型、复合类型等。再比如说，如果从编译器何时能获知类型大小的角度出发，可以分成两类:

- 定长类型( sized )，这些类型的大小在编译时是已知的
- 不定长类型( unsized )，与定长类型相反，它的大小只有到了程序运行时才能动态获知，这种类型又被称之为 DST

### 动态大小类型DST

之前学过的几乎所有类型，都是固定大小的类型（只变量本身的大小，即栈空间大小），包括集合 `Vec`、`String` 和 `HashMap` 等，而动态大小类型是：**编译器无法在编译期得知该类型值的大小，只有到了程序运行时，才能动态获知**。对于动态类型，我们使用 `DST`(dynamically sized types)或者 `unsized` 类型来称呼它。

上述的这些集合虽然底层数据可动态变化，感觉像是动态大小的类型。但是实际上，**这些底层数据只是保存在堆上，在栈中还存有一个引用类型**，该引用包含了集合的内存地址、元素数目、分配空间信息，通过这些信息，编译器对于该集合的实际大小了若指掌，最最重要的是：**栈上的引用类型是固定大小的**，因此它们依然是固定大小的类型。

**正因为编译器无法在编译期获知类型大小，若试图在代码中直接使用 DST 类型，将无法通过编译。**

#### 试图创建动态大小的数组

```rust
fn my_function(n: usize) {
    let array = [123; n];
}
```

以上代码就会报错(错误输出的内容并不是因为 DST，但根本原因是类似的)，因为 `n` 在编译期无法得知，而数组类型的一个组成部分就是长度，长度变为动态的，自然类型就变成了 unsized 。

#### 切片

切片也是一个典型的 DST 类型，具体详情后续在Rust难点公关章节中介绍。

#### str

它既不是 `String` 动态字符串，也不是 `&str` 字符串切片，而是一个 `str`。它是一个动态类型，同时还是 `String` 和 `&str` 的底层数据类型。 由于 `str` 是动态类型，因此它的大小直到运行期才知道，下面的代码会因此报错：

```rust
// error
let s1: str = "Hello there!";
let s2: str = "How's it going?";

// ok
let s3: &str = "on?"
```

Rust 需要明确地知道一个特定类型的变量占据了多少（栈）内存空间，同时该类型的所有变量都必须使用相同大小的内存。

所以，我们只有一条路走，那就是给它们一个固定大小的类型：`&str`。那么为何字符串切片 `&str` 就是固定大小呢？因为它的引用存储在栈上，引用中包含有堆上数据内存地址、长度等信息，这些信息都是固定大小，可以通过这些信息知道堆上数据位置大小。因此可以得出字符串切片是固定大小类型的结论。

与 `&str` 类似，`String` 字符串也是固定大小的类型。

正是因为 `&str` 的引用有了底层堆数据的明确信息，它才是固定大小类型。假设如果它没有这些信息呢？那它也将变成一个动态类型。因此，将动态数据固定化的秘诀就是**使用引用指向这些动态数据，然后在引用中存储相关的内存位置、长度等信息**。

#### 特征对象

```rust
fn foobar_1(thing: &dyn MyThing) {}     // OK
fn foobar_2(thing: Box<dyn MyThing>) {} // OK
fn foobar_3(thing: MyThing) {}          // ERROR!
```

如上所示，只能通过引用或 `Box` 的方式来使用特征对象，直接使用将报错！

#### 总结：只能间接使用的DST

Rust 中常见的 `DST` 类型有: `str`、`[T]`、`dyn Trait`，**它们都无法单独被使用，必须要通过引用或者 `Box` 来间接使用** 。

### Sized特征

既然动态类型的问题这么大，那么在使用泛型时，Rust 如何保证我们的泛型参数是固定大小的类型呢？例如以下泛型函数：

```rust
fn generic<T>(t: T) {
    // --snip--
}
```

在之前的课程章节中，也没有做过任何事情去做相关的限制，那 `T` 怎么就成了固定大小的类型了？奥秘在于编译器自动帮我们加上了 `Sized` 特征约束：

```rust
fn generic<T: Sized>(t: T) {
    // --snip--
}
```

在上面，Rust 自动添加的特征约束 `T: Sized`，表示泛型函数只能用于一切实现了 `Sized` 特征的类型上，而**所有在编译时就能知道其大小的类型，都会自动实现 `Sized` 特征**，几乎所有类型都实现了 `Sized` 特征，除了上面那个坑坑的 `str`，还有特征。

**每一个特征都是一个可以通过名称来引用的动态大小类型**。因此如果想把特征作为具体的类型来传递给函数，你必须将其转换成一个特征对象：诸如 `&dyn Trait` 或者 `Box<dyn Trait>` (还有 `Rc<dyn Trait>`)这些引用类型。

现在还有一个问题：假如想在泛型函数中使用动态数据类型怎么办？可以使用 `?Sized` 特征：

```rust
fn generic<T: ?Sized>(t: &T) {
    // --snip--
}
```

`?Sized` 特征用于表明类型 `T` 既有可能是固定大小的类型，也可能是动态大小的类型。还有一点要注意的是，函数参数类型从 `T` 变成了 `&T`，因为 `T` 可能是动态大小的，因此需要用一个固定大小的指针(引用)来包裹它。

### Box<str>

使用 `Box` 可以将一个动态大小的特征变成一个具有固定大小的特征对象，能否故技重施，将 `str` 封装成一个固定大小类型？

先回想下，章节前面的内容介绍过该如何把一个动态大小类型转换成固定大小的类型： **使用引用指向这些动态数据，然后在引用中存储相关的内存位置、长度等信息**。

根据这个，我们可以推测：首先，`Box<str>` 使用了一个引用来指向 `str`，嗯，满足了第一个条件。但是第二个条件呢？`Box` 中有该 `str` 的长度信息吗？暂时不知道，我们用具体代码来验证：

```rust
let string: Box<str> = Box::new("banana");
```

运行：

```rust
|     let string: Box<str> = Box::new("banana");
|                 --------   ^^^^^^^^^^^^^^^^^^ expected `str`, found `&str`
|                 |
|                 expected due to this
|
= note: expected struct `Box<str>`
found struct `Box<&str>`
```

看上去，`Box::new("banana")`返回的是`Box<&str>`而不是`Box<str>`；也许我们可以解引用字符串字面量来变成`Box<str>`？：

```rust
let string: Box<str> = Box::new(*"banana");
```

运行：

```rust
|     let string: Box<str> = Box::new(*"banana");
|                            -------- ^^^^^^^^^ doesn't have a size known at compile-time
|                            |
|                            required by a bound introduced by this call
|
= help: the trait `Sized` is not implemented for `str`
```

我们无法获得DST类型的所有权，因为其大小未知。解引用会获得指针指向数据的所有权。

直接`Box::new`的方式不行，好在可以通过另外的方式实现：

```rust
let string: Box<str> = String::from("banana").into_boxed_str();
// or
let string: Box<str> = Box::from("banana");
```

通过`from`的形式可以将字符串字面量转变为`Box<str>`，这样就能通过编译。

来看看`Box`类型变量所占内存大小：

```rust
use std::error::Error;

fn main() {
    // Prints 8, 8, 8, 16, 16, 16
    println!(
        "{}, {}, {}, {}, {}, {}",
        std::mem::size_of::<Box<i8>>(),
        std::mem::size_of::<Box<&[i8]>>(),
        std::mem::size_of::<Box<&str>>(),
        std::mem::size_of::<Box<[i8]>>(),
        std::mem::size_of::<Box<str>>(),
        std::mem::size_of::<Box<dyn Error>>(),
    );
}
```

从上面打印结果可以看出，`Box`类型的引用，对于`Sized`类型，其大小为`8`（单纯的一个指针大小），对于DST类型，其大小为`16`，多出来的8字节是什么呢？

在Rust中，一个指针一般来说其大小等于：`size_of::<usize>()`，但对于DST类型的指针是个例外。目前一个`Box<DST>`类型的大小是：`2 * size_of::<usize>()`。一个DST类型的Box指针也叫`胖指针(FatPtr)`。

目前，有两种类型的DST: slices(切片)和traits(特征)。其对应的`胖指针(FatPtr)`定义大致如下：

```rust
struct FatPtr<T> {
    data: *const T,
    len: usize,
}
```

> 注意： 对于trait类型的胖指针，`len`字段替换为指向`vtable`的一个指针。

到此就可以回答：可以通过`Box`封装`str`对象为`Box<str>`，`Box<str>`指针包含有长度信息。

#### Box<str>有什么用？

标准库已经存在`String`对象，和`Box<str>`一样，它们都是具有所有权的变量。并且`String`对象，功能更加强大，既然这样那`Box<str>`存在的必要性是什么呢？

看如下代码：

```rust
fn main() {
    println!("{}", std::mem::size_of::<String>());
    println!("{}", std::mem::size_of::<Box<str>>());
}
```

打印结果：

```rust
24
16
```

可以看出：`Box<str>`比`String`类型，小一个8个字节。`String`存储了：`pointer` + `length` + `capacity`，而`Box<str>`只存储了: `pointer` + `length`。`capacity`可以帮助`String`高效的执行append操作。

在一些罕见的场景可以使用`Box<str>`做一定优化，如当有非常大量的不可变字符串时，标准库中的`interner`就是一个例子：

```rust
pub struct Interner {
    names: HashMap<Box<str>, Symbol>,
    strings: Vec<Box<str>>,
}
```



参考资料：

1. https://betterprogramming.pub/strings-in-rust-28c08a2d3130

2. https://stackoverflow.com/questions/55814114/why-does-boxt-need-16-bytes-in-memory-but-a-referenced-slice-needs-only-8
3. https://users.rust-lang.org/t/use-case-for-box-str-and-string/8295/3



## 枚举和整数转换

### 一个真实场景的需求

在实际场景中，从枚举到整数的转换有时还是非常需要的，例如你有一个枚举类型，然后需要从外面传入一个整数（为什么不传枚举？？因为用户命令行输入项无法传入枚举），用于控制后续的流程走向，此时就需要用整数去匹配相应的枚举(你也可以用整数匹配整数-, -，看看会不会被喷)。

### C语言的实现

对于 C 语言来说，万物皆邪恶，因此我们不讨论安全，只看实现，不得不说很简洁：

```rust
#include <stdio.h>

enum atomic_number {
    HYDROGEN = 1,
    HELIUM = 2,
    // ...
    IRON = 26,
};

int main(void)
{
    enum atomic_number element = 26;

    if (element == IRON) {
        printf("Beware of Rust!\n");
    }

    return 0;
}
```

但是在 Rust 中，以下代码：

```rust
enum MyEnum {
    A = 1,
    B,
    C,
}

fn main() {
    // 将枚举转换成整数，顺利通过
    let x = MyEnum::C as i32;

    // let y: MyEnum = 1 as MyEnum; // 报错：an `as` expression can only be used to convert between primitive types or to coerce to a specific trait object

    // 将整数转换为枚举，失败
    match x {
        MyEnum::A => {}
        MyEnum::B => {}
        MyEnum::C => {}
        _ => {}
    }
}
```

就会报错: `MyEnum::A => {} mismatched types, expected i32, found enum MyEnum`。

### 使用第三方库

 Rust 的生态目前已经发展的很不错，类似的需求总是有的，这里我们先使用`num-traits`和`num-derive`来试试。

在`Cargo.toml`中引入：

```toml
[dependencies]
num-traits = "0.2.14"
num-derive = "0.3.3"
```

代码如下:

```rust
use num_derive::FromPrimitive;
use num_traits::FromPrimitive;

#[derive(FromPrimitive)]
enum MyEnum {
    A = 1,
    B,
    C,
}

fn main() {
    let x = 2;

    match FromPrimitive::from_i32(x) {
        Some(MyEnum::A) => println!("Got A"),
        Some(MyEnum::B) => println!("Got B"),
        Some(MyEnum::C) => println!("Got C"),
        None            => println!("Couldn't convert {}", x),
    }
}
```

除了上面的库，还可以使用一个较新的库: [`num_enums`](https://github.com/illicitonion/num_enum)。

### TryFrom + 宏

在 Rust 1.34 后，可以实现`TryFrom`特征来做转换:

```rust
use std::convert::TryFrom;

impl TryFrom<i32> for MyEnum {
    type Error = ();

    fn try_from(v: i32) -> Result<Self, Self::Error> {
        match v {
            x if x == MyEnum::A as i32 => Ok(MyEnum::A),
            x if x == MyEnum::B as i32 => Ok(MyEnum::B),
            x if x == MyEnum::C as i32 => Ok(MyEnum::C),
            _ => Err(()),
        }
    }
}
```

以上代码定义了从`i32`到`MyEnum`的转换，接着就可以使用`TryInto`来实现转换：

```rust
use std::convert::TryInto;

fn main() {
    let x = MyEnum::C as i32;

    match x.try_into() {
        Ok(MyEnum::A) => println!("a"),
        Ok(MyEnum::B) => println!("b"),
        Ok(MyEnum::C) => println!("c"),
        Err(_) => eprintln!("unknown number"),
    }
}
```

但是上面的代码有个问题，需要为每个枚举成员都实现一个转换分支，非常麻烦。好在可以使用宏来简化，自动根据枚举的定义来实现`TryFrom`特征:

```rust
#[macro_export]
macro_rules! back_to_enum {
    ($(#[$meta:meta])* $vis:vis enum $name:ident {
        $($(#[$vmeta:meta])* $vname:ident $(= $val:expr)?,)*
    }) => {
        $(#[$meta])*
        $vis enum $name {
            $($(#[$vmeta])* $vname $(= $val)?,)*
        }

        impl std::convert::TryFrom<i32> for $name {
            type Error = ();

            fn try_from(v: i32) -> Result<Self, Self::Error> {
                match v {
                    $(x if x == $name::$vname as i32 => Ok($name::$vname),)*
                    _ => Err(()),
                }
            }
        }
    }
}

back_to_enum! {
    enum MyEnum {
        A = 1,
        B,
        C,
    }
}
```



### 邪恶之王 std::mem::transmute

**这个方法原则上并不推荐，但是有其存在的意义，如果要使用，你需要清晰的知道自己为什么使用**。

当你知道数值一定不会超过枚举的范围时(例如枚举成员对应 1，2，3，传入的整数也在这个范围内)，就可以使用这个方法完成变形。

> 最好使用#[repr(..)]来控制底层类型的大小，免得本来需要 i32，结果传入 i64，最终内存无法对齐，产生奇怪的结果

```rust
#[repr(i32)]
enum MyEnum {
    A = 1, B, C
}

fn main() {
    let x = MyEnum::C;
    let y = x as i32;
    let z: MyEnum = unsafe { std::mem::transmute(y) };

    // match the enum that came from an int
    match z {
        MyEnum::A => { println!("Found A"); }
        MyEnum::B => { println!("Found B"); }
        MyEnum::C => { println!("Found C"); }
    }
}
```

### 总结

列举了常用(其实差不多也是全部了，还有一个 unstable 特性没提到)的从整数转换为枚举的方式，推荐度按照出现的先后顺序递减。

但是推荐度最低，不代表它就没有出场的机会，只要使用边界清晰，一样可以大放光彩，例如最后的`transmute`函数。


