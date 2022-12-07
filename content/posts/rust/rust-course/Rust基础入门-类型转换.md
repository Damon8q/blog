---
title: "[Rust course]-10 基础入门-类型转换"
date: 2022-05-06T17:54:00+08:00
lastmod: 2022-05-06T17:54:00+08:00
author: nange
draft: false
description: "Rust集合类型"

categories: ["programming"]
series: ["rust-course"]
series_weight: 10
tags: ["rust"]
---

## 类型转换

Rust 是类型安全的语言，因此在 Rust 中做类型转换不是一件简单的事。

### as转换

常用的转换形式：

```rust
fn main() {
   let a = 3.1 as i8;
   let b = 100_i8 as i32;
   let c = 'a' as u8; // 将字符'a'转换为整数，97

   println!("{},{},{}",a,b,c)
}
```

> 使用类型转换需要小心，因为如果执行以下操作 `300_i32 as i8`，你将获得 `44` 这个值，而不是 `300`，因为 `i8` 类型能表达的的最大值为 `2^7 - 1`，使用以下代码可以查看 `i8` 的最大值：

```rust
let a = i8::MAX;
println!("{}",a);
```

#### 内存地址转换为指针

```rust
let mut values: [i32; 2] = [1, 2];
let p1: *mut i32 = values.as_mut_ptr();
let first_address = p1 as usize; // 将p1内存地址转换为一个整数
let second_address = first_address + 4; // 4 == std::mem::size_of::<i32>()，i32类型占用4个字节，因此将内存地址 + 4
let p2 = second_address as *mut i32; // 访问该地址指向的下一个整数p2
unsafe {
    *p2 += 1;
}
assert_eq!(values[1], 3);
```

#### 强制类型转换的边角知识

1. 转换不具有传递性 就算 `e as U1 as U2` 是合法的，也不能说明 `e as U2` 是合法的（`e` 不能直接转换成 `U2`）。

### TryInto转换

在一些场景中，使用 `as` 关键字会有比较大的限制。如果你想要在类型转换上拥有完全的控制而不依赖内置的转换，例如处理转换错误，那么可以使用 `TryInto` ：

```rust
use std::convert::TryInto;

fn main() {
   let a: u8 = 10;
   let b: u16 = 1500;

   let b_: u8 = b.try_into().unwrap();

   if a < b_ {
     println!("Ten is less than one hundred.");
   }
}
```

上面代码中引入了 `std::convert::TryInto` 特征，但是却没有使用它，可能有些困惑，主要原因在于**如果要使用一个特征的方法，那么你需要引入该特征到当前的作用域中**，在上面用到了 `try_into` 方法，因此需要引入对应的特征。但是 Rust 又提供了一个非常便利的办法，把最常用的标准库中的特征通过[`std::prelude`](https://course.rs/appendix/prelude.html)模块提前引入到当前作用域中，其中包括了 `std::convert::TryInto`，因此也可以删除上面的`use`项。

`try_into` 会尝试进行一次转换，并返回一个 `Result`，此时就可以对其进行相应的错误处理。由于我们的例子只是为了快速测试，因此使用了 `unwrap` 方法。

最主要的是 `try_into` 转换会捕获大类型向小类型转换时导致的溢出错误：

```rust
fn main() {
    let b: i16 = 1500;

    let b_: u8 = match b.try_into() {
        Ok(b1) => b1,
        Err(e) => {
            println!("{:?}", e.to_string());
            0
        }
    };
}
```

运行后输出如下 `"out of range integral type conversion attempted"`，在这里我们程序捕获了错误，编译器告诉我们类型范围超出的转换是不被允许的，因为我们试图把 `1500_i16` 转换为 `u8` 类型，后者明显不足以承载这么大的值。

### 通用类型转换

虽然 `as` 和 `TryInto` 很强大，**但其只能应用在数值类型上**。可是 Rust 有如此多的类型，想要为这些类型实现转换，需要另谋出路，先来看看在一个笨办法，将一个结构体转换为另外一个结构体：

```rust
struct Foo {
    x: u32,
    y: u16,
}

struct Bar {
    a: u32,
    b: u16,
}

fn reinterpret(foo: Foo) -> Bar {
    let Foo { x, y } = foo;
    Bar { a: x, b: y }
}
```

简单粗暴，但是从另外一个角度来看，也挺啰嗦的，好在 Rust 为我们提供了更通用的方式来完成这个目的。

#### 强制类型转换

在匹配特征时，不会做任何强制转换(除了方法)。一个类型 `T` 可以强制转换为 `U`，不代表 `impl T` 可以强制转换为 `impl U`，例如下面的代码就无法通过编译检查：

```rust
trait Trait {}

fn foo<X: Trait>(t: X) {}

// 这里也可以不加生命周期泛型参数
// 至于加上是什么含义，暂时还不懂，后面学到了再来补充
impl<'a> Trait for &'a i32 {}

fn main() {
    let t: &mut i32 = &mut 0;
    foo(t);
}
```

报错如下：

```rust
error[E0277]: the trait bound `&mut i32: Trait` is not satisfied
--> src/main.rs:9:9
|
9 |     foo(t);
|         ^ the trait `Trait` is not implemented for `&mut i32`
|
= help: the following implementations were found:
        <&'a i32 as Trait>
= note: `Trait` is implemented for `&i32`, but not for `&mut i32`
```

`&i32` 实现了特征 `Trait`， `&mut i32` 可以转换为 `&i32`，但是 `&mut i32` 依然无法作为 `Trait` 来使用。

#### 点操作符

方法调用的点操作符看起来简单，实际上非常不简单，它在调用时，会发生很多魔法般的类型转换，例如：自动引用、自动解引用，强制类型转换直到类型能匹配等。

假设有一个方法 `foo`，它有一个接收器(接收器就是 `self`、`&self`、`&mut self` 参数)。如果调用 `value.foo()`，编译器在调用 `foo` 之前，需要决定到底使用哪个 `Self` 类型来调用。现在假设 `value` 拥有类型 `T`：

1. 首先，编译器检查它是否可以直接调用 `T::foo(value)`，称之为**值方法调用**

2. 如果上一步调用无法完成(例如方法类型错误或者特征没有针对 `Self` 进行实现，上文提到过特征不能进行强制转换)，那么编译器会尝试增加自动引用，例如会尝试以下调用： `<&T>::foo(value)` 和 `<&mut T>::foo(value)`，称之为**引用方法调用**

3. 若上面两个方法依然不工作，编译器会试着解引用 `T` ，然后再进行尝试。这里使用了 `Deref` 特征 —— 若 `T: Deref<Target = U>` (`T` 可以被解引用为 `U`)，那么编译器会使用 `U` 类型进行尝试，称之为**解引用方法调用**

4. 若 `T` 不能被解引用，且 `T` 是一个定长类型(在编译器类型长度是已知的)，那么编译器也会尝试将 `T` 从定长类型转为不定长类型，例如将 `[i32; 2]` 转为 `[i32]`

5. 若还是不行，那...没有那了，最后编译器大喊一声：汝欺我甚，不干了！

下面我们来用一个例子来解释上面的方法查找算法:

```rust
let array: Rc<Box<[T; 3]>> = ...;
let first_entry = array[0];
```

`array` 数组的底层数据隐藏在了重重封锁之后，那么编译器如何使用 `array[0]` 这种数组原生访问语法通过重重封锁，准确的访问到数组中的第一个元素？

1. 首先， `array[0]` 只是[`Index`](https://doc.rust-lang.org/std/ops/trait.Index.html)特征的语法糖：编译器会将 `array[0]` 转换为 `array.index(0)` 调用，当然在调用之前，编译器会先检查 `array` 是否实现了 `Index` 特征。

2. 接着，编译器检查 `Rc<Box<[T; 3]>>` 是否有否实现 `Index` 特征，结果是否，不仅如此，`&Rc<Box<[T; 3]>>` 与 `&mut Rc<Box<[T; 3]>>` 也没有实现。

3. 上面的都不能工作，编译器开始对 `Rc<Box<[T; 3]>>` 进行解引用，把它转变成 `Box<[T; 3]>`

4. 此时继续对 `Box<[T; 3]>` 进行上面的操作 ：`Box<[T; 3]>`， `&Box<[T; 3]>`，和 `&mut Box<[T; 3]>` 都没有实现 `Index` 特征，所以编译器开始对 `Box<[T; 3]>` 进行解引用，然后我们得到了 `[T; 3]`

5. `[T; 3]` 以及它的各种引用都没有实现 `Index` 索引(是不是很反直觉:D，在直觉中，数组都可以通过索引访问，实际上只有数组切片才可以!)，它也不能再进行解引用，因此编译器只能祭出最后的大杀器：将定长转为不定长，因此 `[T; 3]` 被转换成 `[T]`，也就是数组切片，它实现了 `Index` 特征，因此最终我们可以通过 `index` 方法访问到对应的元素。

再来看看以下更复杂的例子：

```rust
fn do_stuff<T: Clone>(value: &T) {
    let cloned = value.clone();
}
```

上面例子中 `cloned` 的类型是什么？首先编译器检查能不能进行**值方法调用**， `value` 的类型是 `&T`，同时 `clone` 方法的签名也是 `&T` ： `fn clone(&T) -> T`，因此可以进行值方法调用，再加上编译器知道了 `T` 实现了 `Clone`，因此 `cloned` 的类型是 `T`。

如果 `T: Clone` 的特征约束被移除呢？

```rust
fn do_stuff<T>(value: &T) {
    let cloned = value.clone();
}
```

首先，从直觉上来说，该方法会报错，因为 `T` 没有实现 `Clone` 特征，但是真实情况是什么呢？

我们先来推导一番。 首先通过值方法调用就不再可行，因为 `T` 没有实现 `Clone` 特征，也就无法调用 `T` 的 `clone` 方法。接着编译器尝试**引用方法调用**，此时 `T` 变成 `&T`，在这种情况下， `clone` 方法的签名如下： `fn clone(&&T) -> &T`，接着我们现在对 `value` 进行了引用。 编译器发现 `&T` 实现了 `Clone` 类型(所有的引用类型都可以被复制，因为其实就是复制一份地址)，因此可以推出 `cloned` 也是 `&T` 类型。

最终，我们复制出一份引用指针，这很合理，因为值类型 `T` 没有实现 `Clone`，只能去复制一个指针了。

#### 变形记（Transmutes）

Transmutes非常不安全，一般不推荐使用。

1. `mem::transmute<T, U>` 将类型 `T` 直接转成类型 `U`，唯一的要求就是，这两个类型占用同样大小的字节数！

2. `mem::transmute_copy<T, U>` 它更加危险和不安全。它从 `T` 类型中拷贝出 `U` 类型所需的字节数，然后转换成 `U`。`mem::transmute` 尚有大小检查，能保证两个数据的内存大小一致，现在这哥们干脆连这个也丢了，只不过 `U` 的尺寸若是比 `T` 大，会是一个未定义行为。

列举两个有用的 `transmute` 应用场景：

* 将裸指针变成函数指针：
  
  ```rust
  fn foo() -> i32 {
      0
  }
  
  let pointer = foo as *const ();
  let function = unsafe { 
      // 将裸指针转换为函数指针
      std::mem::transmute::<*const (), fn() -> i32>(pointer) 
  };
  assert_eq!(function(), 0);
  ```

* 延长生命周期，或者缩短一个静态生命周期寿命：
  
  ```rust
  struct R<'a>(&'a i32);
  
  // 将 'b 生命周期延长至 'static 生命周期
  unsafe fn extend_lifetime<'b>(r: R<'b>) -> R<'static> {
      std::mem::transmute::<R<'b>, R<'static>>(r)
  }
  
  // 将 'static 生命周期缩短至 'c 生命周期
  unsafe fn shorten_invariant_lifetime<'b, 'c>(r: &'b mut R<'static>) -> &'b mut R<'c> {
      std::mem::transmute::<&'b mut R<'static>, &'b mut R<'c>>(r)
  }
  ```

以上例子非常先进！但是是非常不安全的 Rust 行为！
