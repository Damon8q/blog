---
title: "[Rust course]-12 基础入门-包和模块"
date: 2022-05-12T21:23:00+08:00
lastmod: 2022-05-12T21:23:00+08:00
author: nange
draft: false
description: "Rust包和模块"

categories: ["programming"]
series: ["rust-course"]
series_weight: 12
tags: ["rust"]
---

## 包和模块

Rust提供了如下几个概念来进行代码组织管理:

* 项目(Packages)： 一个 `Cargo` 提供的 `feature`，可以用来构建、测试和分享包

* 工作空间(Workspace)：对于大型项目，可以进一步将多个包联合在一起，组织成工作空间

* 包(Crate)：一个由多个模块组成的树形结构，可以作为三方库进行分发，也可以生成可执行文件进行运行

* 模块(Module)：可以一个文件多个模块，也可以一个文件一个模块，模块可以被认为是真实项目中的代码组织单元

## 包 Crate 和 项目 Package 的区别

由于 `Package` 就是一个项目，因此它包含有独立的 `Cargo.toml` 文件，以及因为功能性被组织在一起的一个或多个包。一个 `Package` 只能包含**一个**库(library)类型的包，但是可以包含**多个**二进制可执行类型的包。

### 二进制 Package

创建二进制 `Package`:

```shell
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```

Cargo 有一个惯例：**`src/main.rs` 是二进制包的根文件，该二进制包的包名跟所属 `Package` 相同，在这里都是 `my-project`**，所有的代码执行都从该文件中的 `fn main()` 函数开始。

### 库Package

创建库类型`Package`:

```shell
$ cargo new my-lib --lib
     Created library `my-lib` package
$ ls my-lib
Cargo.toml
src
$ ls my-lib/src
lib.rs
```

与 `src/main.rs` 一样，Cargo 知道，如果一个 `Package` 包含有 `src/lib.rs`，意味它包含有一个库类型的同名包 `my-lib`，该包的根文件是 `src/lib.rs`。

### 易混淆的Package和包

`Package`和包之所以易混淆，是因为用`cargo new`创建的`Package`和它其中包含的包是同名的。不过只要牢记 `Package` 是一个项目工程，而包只是一个编译单元，基本上也就不会混淆这个两个概念了：`src/main.rs` 和 `src/lib.rs` 都是编译单元，因此它们都是包。

### 典型的Package 结构

如果一个 `Package` 同时拥有 `src/main.rs` 和 `src/lib.rs`，那就意味着它包含两个包：库包和二进制包，这两个包名也都是 `my-project` —— 都与 `Package` 同名。

一个真实项目中典型的 `Package`，可能会包含多个二进制包，这些包文件被放在 `src/bin` 目录下，每一个文件都是独立的二进制包，同时也可以包含一个库包，该包只能存在一个 `src/lib.rs`：

```text
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── main.rs
│   ├── lib.rs
│   └── bin
│       └── main1.rs
│       └── main2.rs
├── tests
│   └── some_integration_tests.rs
├── benches
│   └── simple_bench.rs
└── examples
    └── simple_example.rs
```

* 唯一库包：`src/lib.rs`
* 默认二进制包：`src/main.rs`，编译后生成的可执行文件与 `Package` 同名
* 其余二进制包：`src/bin/main1.rs` 和 `src/bin/main2.rs`，它们会分别生成一个文件同名的二进制可执行文件
* 集成测试文件：`tests` 目录下
* 基准性能测试 `benchmark` 文件：`benches` 目录下
* 项目示例：`examples` 目录下

## 模块 Module

模块是Rust代码的构成单元。使用模块可以将代码按功能进行组织。

### 创建嵌套模块

`cargo new --lib restaurant` 创建一个库类型的`Package`，将一下代码放入`src/lib.rs`中：

```rust
// 餐厅前厅，用于吃饭
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

以上的代码创建了三个模块，有几点需要注意的：

* 使用 `mod` 关键字来创建新模块，后面紧跟着模块名称
* 模块可以嵌套，这里嵌套的原因是招待客人和服务都发生在前厅，因此我们的代码模拟了真实场景
* 模块中可以定义各种 Rust 类型，例如函数、结构体、枚举、特征等
* 所有模块均定义在同一个文件中

类似上述代码中所做的，使用模块，我们就能将功能相关的代码组织到一起，方便自己和他人理解及使用。

### 模块树

`src/main.rs` 和 `src/lib.rs` 被称为crate root，因为这两个文件的内容形成了一个模块 `crate`，该模块位于包的树形结构(由模块组成的树形结构)的根部：

```text
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

这颗树展示了模块之间**彼此的嵌套**关系，因此被称为**模块树**。其中 `crate` 是 `src/lib.rs` 文件，文件中的三个模块分别形成了模块树的剩余部分。

#### 父子模块

如果模块 `A` 包含模块 `B`，那么 `A` 是 `B` 的父模块，`B` 是 `A` 的子模块。在上例中，`front_of_house` 是 `hosting` 和 `serving` 的父模块，反之，后两者是前者的子模块。在 Rust 中，我们也通过路径的方式来引用模块。

### 用路径引用模块

想要调用一个函数，就需要知道它的路径，在 Rust 中，这种路径有两种形式：

* **绝对路径**，从包根开始，路径名以包名或者 `crate` 作为开头
* **相对路径**，从当前模块开始，以 `self`，`super` 或当前模块的标识符作为开头

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();

    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

如果不确定使用那种路径形式更好，可优先使用绝对路径。

### 代码可见性

我们运行上面的代码，会得到如下报错：

```rust
error[E0603]: module `hosting` is private
 --> src/lib.rs:9:28
  |
9 |     crate::front_of_house::hosting::add_to_waitlist();
  |                            ^^^^^^^ private module
```

错误信息很清晰：`hosting` 模块是私有的，无法在包根进行访问，那么为何 `front_of_house` 模块就可以访问？因为它和 `eat_at_restaurant` 同属于一个作用域内，同一个模块内的代码自然不存在私有化问题。

Rust 出于安全的考虑，默认情况下，所有的类型都是私有化的，包括函数、方法、结构体、枚举、常量，是的，就连模块本身也是私有化的。

在 Rust 中，**父模块完全无法访问子模块中的私有项，但是子模块却可以访问父模块、父父..模块的私有项**。

#### pub关键字

Rust 提供了 `pub` 关键字，通过它你可以控制模块和模块中指定项的可见性。将代码改为如下：

```rust
mod front_of_house {
    pub mod hosting {
        fn add_to_waitlist() {}
    }
}

/*--- snip ----*/
```

但是不幸的是，又报错了：

```rust
error[E0603]: function `add_to_waitlist` is private
  --> src/lib.rs:12:30
   |
12 |     front_of_house::hosting::add_to_waitlist();
   |                              ^^^^^^^^^^^^^^^ private function
```

模块可见性不代表模块内部项的可见性，模块的可见性仅仅是允许其它模块去引用它，但是想要引用它内部的项，还得继续将对应的项标记为 `pub`。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

/*--- snip ----*/
```

这样代码就能顺利编译了。

### 使用super引用模块

```rust
fn serve_order() {}

// 厨房模块
mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

这里也可以使用`crate::serve_order` 的方式。如果你确定未来这种层级关系不会改变，那么 `super::serve_order` 的方式会更稳定。所以路径的选用，往往还是取决于场景，以及未来代码的可能走向。

### 使用self引用模块

```rust
fn serve_order() {
    self::back_of_house::cook_order()
}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        crate::serve_order();
    }

    pub fn cook_order() {}
}
```

`self` 其实就是引用自身模块中的项，而因为本就是自身模块，因此往往`self`是多次一举的。完全可以直接调用 `back_of_house`，但是 `self` 还有一个大用处，在下一节中我们会讲。

### 结构体和枚举的可见性

为何要把结构体和枚举的可见性单独拎出来讲呢？因为这两个家伙的成员字段拥有完全不同的可见性：

* 将结构体设置为 `pub`，但它的所有字段依然是私有的
* 将枚举设置为 `pub`，它的所有字段也将对外可见

原因在于，枚举和结构体的使用方式不一样。如果枚举的成员对外不可见，那该枚举将一点用都没有，因此枚举成员的可见性自动跟枚举可见性保持一致，这样可以简化用户的使用。

而结构体使用场景复杂，那索性就设置为全部不可见，将选择权交给程序员。

### 模块与文件分离

在之前的例子中，我们所有的模块都定义在 `src/lib.rs` 中，但是当模块变多或者变大时，需要将模块放入一个单独的文件中，让代码更好维护（实际项目不可能都放在lib.rs）。

把 `front_of_house` 前厅分离出来，放入一个单独的文件中 `src/front_of_house.rs`：

```rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

然后，将以下代码留在 `src/lib.rs` 中：

```rust
mod front_of_house;

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

有几点值得注意：

* `mod front_of_house;` 告诉 Rust 从另一个和模块 `front_of_house` 同名的文件中加载该模块的内容
* 使用绝对路径的方式来引用 `hosting` 模块：`crate::front_of_house::hosting;`

通过 `mod front_of_house;` 这条声明语句从该文件中把模块内容加载进来。可以认为，模块 `front_of_house` 的定义还是在 `src/lib.rs` 中，只不过模块的具体内容被移动到了 `src/front_of_house.rs` 文件中。

在这里出现了一个新的关键字 `use`，联想到其它章节我们见过的标准库引入 `use std::fmt;`，该关键字用来将外部模块中的项引入到当前作用域中来，这样无需冗长的父模块前缀即可调用：`hosting::add_to_waitlist();`，在下节中，我们将对 `use` 进行详细的讲解。

## 使用use及受限可见性

在 Rust 中，可以使用 `use` 关键字把路径提前引入到当前作用域中，随后的调用就可以省略该路径，极大地简化了代码。

### 基本引入方式

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting; // 绝对路径形式
// use front_of_house::hosting::add_to_waitlist; // 相对路径形式

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    // add_to_waitlist(); 相对路径形式
}
```

#### 引入模块还是函数

从使用简洁性来说，引入函数更好，但是在某些时候，引入模块会更好：

* 需要引入同一个模块的多个函数
* 作用域中存在同名函数

因此：**优先使用最细粒度(引入函数、结构体等)的引用方式，如果引起了某种麻烦(例如前面两种情况)，再使用引入模块的方式**。

### 避免同名引用

#### 模块::函数

```rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
}

fn function2() -> io::Result<()> {
    // --snip--
}
```

可以看出，避免同名冲突的关键，就是使用**父模块的方式来调用**，除此之外，还可以给予引入的项起一个别名。

#### as别名引用

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

使用 `as` 给予它一个全新的名称 `IoResult`，这样就不会再产生冲突。

### 引入项再导出

当外部的模块项 `A` 被引入到当前模块中时，它的可见性自动被设置为私有的，如果你希望允许其它外部代码引用我们的模块项 `A`，那么可以对它进行再导出：

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

如上，使用 `pub use` 即可实现。这里 `use` 代表引入 `hosting` 模块到当前作用域，`pub` 表示将该引入的内容再度设置为可见。

当希望将内部的实现细节隐藏起来或者按照某个目的组织代码时，可以使用 `pub use` 再导出，例如统一使用一个模块来提供对外的 API，那该模块就可以引入其它模块中的 API，然后进行再导出，最终对于用户来说，所有的 API 都是由一个模块统一提供的。

### 使用第三方包

之前我们一直在引入标准库模块或者自定义模块，现在来引入下第三方包中的模块，操作步骤：

1. 修改 `Cargo.toml` 文件，在 `[dependencies]` 区域添加一行：`rand = "0.8.3"`
2. 此时，如果使用的IDE是 `VSCode` 和 `rust-analyzer` 插件，该插件会自动拉取该库，可能需要等它完成后，再进行下一步（VSCode 左下角有提示）

此时，`rand` 包已经被我们添加到依赖中，下一步就是在代码中使用：

```rust
use rand::Rng;

fn main() {
    let secret_number = rand::thread_rng().gen_range(1..101);
}
```

#### crates.io, lib.rs

Rust 社区已经为我们贡献了大量高质量的第三方包，可以在 `crates.io` 或者 `lib.rs` 中检索和使用，从目前来说查找包更推荐 `lib.rs`，搜索功能更强大，内容展示也更加合理，但是下载依赖包还是得用`crates.io`。

### 使用{}简化引入方式

对于以下一行一行的引入方式：

```rust
use std::collections::HashMap;
use std::collections::BTreeMap;
use std::collections::HashSet;

use std::cmp::Ordering;
use std::io;
```

也可以修改为：

```rust
use std::collections::{HashMap,BTreeMap,HashSet};
use std::{cmp::Ordering, io};
```

这样可以减少大量 `use` 的使用。

对于下面的同时引入模块和模块中的项：

```rust
use std::io;
use std::io::Write;
```

也可以使用 `{}` 的方式进行简化:

```rust
use std::io::{self, Write};
```

上面使用到了模块章节提到的 `self` 关键字，用来替代模块自身，结合上一节中的 `self`，可以得出它在模块中的两个用途：

* `use self::xxx`，表示加载当前模块中的 `xxx`。此时 `self` 可省略
* `use xxx::{self, yyy}`，表示，加载当前路径下模块 `xxx` 本身，以及模块 `xxx` 下的 `yyy`

### 使用 * 引入模块下的所有项

对于之前一行一行引入 `std::collections` 的方式，我们还可以使用:

```rust
use std::collections::*;
```

以上这种方式来引入 `std::collections` 模块下的所有公共项，这些公共项自然包含了 `HashMap`，`HashSet` 等想手动引入的集合类型。

当使用 `*` 来引入的时候要格外小心，因为你很难知道到底哪些被引入到了当前作用域中，有哪些会和你自己程序中的名称相冲突。

在实际项目中，这种引用方式往往用于快速写测试代码，它可以把所有东西一次性引入到 `tests` 模块中。

### 受限的可见性

在 Rust 中，包是一个模块树，我们可以通过 `pub(crate) item;` 这种方式来实现：`item` 虽然是对外可见的，但是只在当前包内可见，外部包无法引用到该 `item`。

#### 限制可见语法

`pub(crate)` 或 `pub(in crate::a)` 就是限制可见性语法，前者是限制在整个包内可见，后者是通过绝对路径，限制在包内的某个模块内可见，总结一下：

* `pub` 意味着可见性无任何限制
* `pub(crate)` 表示在当前包可见（及同一个父级模块或者祖先模块都可以）
* `pub(self)` 在当前模块可见
* `pub(super)` 在父模块可见
* `pub(in <path>)` 表示在某个路径代表的模块中可见，其中 `path` 必须是父模块或者祖先模块

#### 一个综合例子

```rust
// 一个名为 `my_mod` 的模块
mod my_mod {
    // 模块中的项默认具有私有的可见性
    fn private_function() {
        println!("called `my_mod::private_function()`");
    }

    // 使用 `pub` 修饰语来改变默认可见性。
    pub fn function() {
        println!("called `my_mod::function()`");
    }

    // 在同一模块中，项可以访问其它项，即使它是私有的。
    pub fn indirect_access() {
        print!("called `my_mod::indirect_access()`, that\n> ");
        private_function();
    }

    // 模块也可以嵌套
    pub mod nested {
        pub fn function() {
            println!("called `my_mod::nested::function()`");
        }

        #[allow(dead_code)]
        fn private_function() {
            println!("called `my_mod::nested::private_function()`");
        }

        // 使用 `pub(in path)` 语法定义的函数只在给定的路径中可见。
        // `path` 必须是父模块（parent module）或祖先模块（ancestor module）
        pub(in crate::my_mod) fn public_function_in_my_mod() {
            print!("called `my_mod::nested::public_function_in_my_mod()`, that\n > ");
            public_function_in_nested()
        }

        // 使用 `pub(self)` 语法定义的函数则只在当前模块中可见。
        pub(self) fn public_function_in_nested() {
            println!("called `my_mod::nested::public_function_in_nested");
        }

        // 使用 `pub(super)` 语法定义的函数只在父模块中可见。
        pub(super) fn public_function_in_super_mod() {
            println!("called my_mod::nested::public_function_in_super_mod");
        }

        pub(crate) fn public_test() {
            println!("public test");
        }
    }

    pub fn call_public_function_in_my_mod() {
        print!("called `my_mod::call_public_funcion_in_my_mod()`, that\n> ");
        nested::public_function_in_my_mod();
        print!("> ");
        nested::public_function_in_super_mod();
    }

    // `pub(crate)` 使得函数只在当前包中可见
    pub(crate) fn public_function_in_crate() {
        println!("called `my_mod::public_function_in_crate()");
    }

    // 嵌套模块的可见性遵循相同的规则
    mod private_nested {
        use crate::my_mod::nested::public_test;
        #[allow(dead_code)]
        pub fn function() {
            super::public_function_in_crate(); // 这样是OK的
            public_test(); // 这样也是OK的
            println!("called `my_mod::private_nested::function()`");
        }
    }
}

fn function() {
    println!("called `function()`");
}

fn main() {
    // 模块机制消除了相同名字的项之间的歧义。
    function();
    my_mod::function();

    // 公有项，包括嵌套模块内的，都可以在父模块外部访问。
    my_mod::indirect_access();
    my_mod::nested::function();
    my_mod::call_public_function_in_my_mod();

    // pub(crate) 项可以在同一个 crate 中的任何地方访问
    my_mod::public_function_in_crate();

    // pub(in path) 项只能在指定的模块中访问
    // 报错！函数 `public_function_in_my_mod` 是私有的
    //my_mod::nested::public_function_in_my_mod();
    // 试一试 ^ 取消该行的注释

    // 模块的私有项不能直接访问，即便它是嵌套在公有模块内部的

    // 报错！`private_function` 是私有的
    // my_mod::private_function();
    // 试一试 ^ 取消此行注释

    // 报错！`private_function` 是私有的
    //my_mod::nested::private_function();
    // 试一试 ^ 取消此行的注释

    // 报错！ `private_nested` 是私有的
    //my_mod::private_nested::function();
    // 试一试 ^ 取消此行的注释
}

```
