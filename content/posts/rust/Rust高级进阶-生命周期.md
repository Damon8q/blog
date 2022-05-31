---
title: "[Rust course]-15 高级进阶-生命周期"
date: 2022-05-16T11:35:00+08:00
lastmod: 2022-05-16T11:35:00+08:00
author: nange
draft: false
description: "Rust生命周期"

categories: ["rust"]
series: ["rust-course"]
series_weight: 15
tags: ["rust-notes"]
---

## 认识生命周期

生命周期，简而言之就是引用的有效作用域。标注生命周期的目的在于，让编译器知道一个未知的引用类型变量（如函数返回值）的有效作用域是什么，避免出现悬垂引用的内存问题。在多数时候，我们不需要指定引用类型变量的生命周期，因为编译器能自动推导，当编译器无法自动推导时，必须手动指定，不然则出现编译报错。

### 悬垂指针和生命周期

生命周期的主要作用是避免悬垂引用：

```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```

此处 `r` 就是一个悬垂指针，它引用了提前被释放的变量 `x`，可以预料到，这段代码会报错：

```rust
error[E0597]: `x` does not live long enough // `x` 活得不够久
  --> src/main.rs:7:17
   |
7  |             r = &x;
   |                 ^^ borrowed value does not live long enough // 被借用的 `x` 活得不够久
8  |         }
   |         - `x` dropped here while still borrowed // `x` 在这里被丢弃，但是它依然还在被借用
9  |
10 |         println!("r: {}", r);
   |                           - borrow later used here // 对 `x` 的借用在此处被使用
```



### 借用检查

为了保证 Rust 的所有权和借用的正确性，Rust 使用了一个借用检查器(Borrow checker)，来检查我们程序的借用正确性：

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

这里，`r` 变量被赋予了生命周期 `'a`，`x` 被赋予了生命周期 `'b`，从图示上可以明显看出生命周期 `'b` 比 `'a` 小很多。

在编译期，Rust 会比较两个变量的生命周期，结果发现 `r` 明明拥有生命周期 `'a`，但是却引用了一个小得多的生命周期 `'b`，在这种情况下，编译器会认为我们的程序存在风险，因此拒绝运行。

如果想要编译通过，也很简单，只要 `'b` 比 `'a` 大就好。总之，`x` 变量只要比 `r` 活得久，那么 `r` 就能随意引用 `x` 且不会存在危险：

```rust
{
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```



### 函数中的生命周期

先来考虑一个例子 - 返回两个字符串切片中较长的那个，该函数的参数是两个字符串切片，返回值也是字符串切片：

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

但是编译报错：

```rust
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter // 参数需要一个生命周期
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is
  borrowed from `x` or `y`
  = 帮助： 该函数的返回值是一个引用类型，但是函数签名无法说明，该引用是借用自 `x` 还是 `y`
help: consider introducing a named lifetime parameter // 考虑引入一个生命周期
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ^^^^    ^^^^^^^     ^^^^^^^     ^^^
```

主要是编译器无法知道该函数的返回值到底引用 `x` 还是 `y` ，**因为编译器需要知道这些，来确保函数调用后的引用生命周期分析**。

就这个函数而言，我们也不知道返回值到底引用哪个，因为一个分支返回 `x`，另一个分支返回 `y`...这可咋办？先来分析下。

编译器的借用检查也无法推导出返回值的生命周期，因为它不知道 `x` 和 `y` 的生命周期跟返回值的生命周期之间的关系是怎样的(说实话，人都搞不清，何况编译器这个大聪明)。因此，这时就回到了文章开头说的内容：在存在多个引用时，编译器有时会无法自动推导生命周期，此时就需要我们手动去标注，通过为参数标注合适的生命周期来帮助编译器进行借用检查的分析。



### 生命周期标注语法

生命周期标注并不会改变任何引用的实际作用域。**标记的生命周期只是为了取悦编译器，让编译器不要难为我们**。

例如一个变量，只能活一个花括号，那么就算你给它标注一个活全局的生命周期，它还是会在前面的花括号结束处被释放掉，并不会真的全局存活。

```rust
&i32        // 一个引用
&'a i32     // 具有显式生命周期的引用
&'a mut i32 // 具有显式生命周期的可变引用
```

`'a`只是一种习惯性写法，也可以是`'b`, `'c`等。

一个生命周期标注，它自身并不具有什么意义，因为生命周期的作用就是告诉编译器多个引用之间的关系：

```rust
fn useless<'a>(first: &'a i32, second: &'a i32) {}
```

此处生命周期标注仅仅说明:**这两个参数 `first` 和 `second` 至少活得和'a 一样久，至于到底活多久或者哪个活得更久，抱歉我们都无法得知**。

#### 函数签名中的生命周期标注

继续之前的 `longest` 函数，从两个字符串切片中返回较长的那个：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

需要注意的点如下：

- 和泛型一样，使用生命周期参数，需要先声明 `<'a>`
- `x`、`y` 和返回值至少活得和 `'a` 一样久(因为返回值要么是 `x`，要么是 `y`)

实际上，这意味着返回值的生命周期与参数生命周期中的较小值一致：虽然两个参数的生命周期都是标注了 `'a`，但是实际上这两个参数的真实生命周期可能是不一样的。

当把具体的引用传给 `longest` 时，那生命周期 `'a` 的大小就是 `x` 和 `y` 的作用域的重合部分（取交集），换句话说，`'a` 的大小将等于 `x` 和 `y` 中较小的那个。由于返回值的生命周期也被标记为 `'a`，因此返回值的生命周期也是 `x` 和 `y` 中作用域较小的那个。

再来看一个例子，该例子证明了 `result` 的生命周期必须等于两个参数中生命周期较小的那个:

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

编译报错：

```rust
error[E0597]: `string2` does not live long enough
 --> src/main.rs:6:44
  |
6 |         result = longest(string1.as_str(), string2.as_str());
  |                                            ^^^^^^^ borrowed value does not live long enough
7 |     }
  |     - `string2` dropped here while still borrowed
8 |     println!("The longest string is {}", result);
  |                                          ------ borrow later used here
```

作为人类，我们可以很清晰的看出 `result` 实际上引用了 `string1`，因为 `string1` 的长度明显要比 `string2` 长，既然如此，编译器不该如此矫情才对，它应该能认识到 `result` 没有引用 `string2`，让我们这段代码通过。只能说，作为尊贵的人类，编译器的发明者，你高估了这个工具的能力，它真的做不到！而且 Rust 编译器在调教上是非常保守的：当可能出错也可能不出错时，它会选择前者，抛出编译错误。

#### 深入思考生命周期标注

使用生命周期的方式往往取决于函数的功能，例如之前的 `longest` 函数，如果它永远只返回第一个参数 `x`，生命周期的标注该如何修改？

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

在此例中，`y` 完全没有被使用，因此 `y` 的生命周期与 `x` 和返回值的生命周期没有任何关系，意味着我们也不必再为 `y` 标注生命周期，只需要标注 `x` 参数和返回值即可。

**函数的返回值如果是一个引用类型，那么它的生命周期只会来源于**：

- 函数参数的生命周期
- 函数体中某个新建引用的生命周期

若是后者情况，就是典型的悬垂引用场景：

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

很显然，该函数会报错：

```rust
error[E0515]: cannot return value referencing local variable `result` // 返回值result引用了本地的变量
  --> src/main.rs:11:5
   |
11 |     result.as_str()
   |     ------^^^^^^^^^
   |     |
   |     returns a value referencing data owned by the current function
   |     `result` is borrowed here
```

主要问题就在于，`result` 在函数结束后就被释放，但是在函数结束后，对 `result` 的引用依然在继续。在这种情况下，没有办法指定合适的生命周期来让编译通过，因此我们也就在 Rust 中避免了悬垂引用。

处理这种情况，最好的办法就是返回内部字符串的所有权，然后把字符串的所有权转移给调用者：

```rust
fn longest(_x: &str, _y: &str) -> String {
    String::from("really long string")
}

fn main() {
    let s = longest("not", "important");
}
```

生命周期语法用来将函数的多个引用参数和返回值的作用域关联到一起，一旦关联到一起后，Rust 就拥有充分的信息来确保我们的操作是内存安全的。



### 结构体中的生命周期

不仅仅函数具有生命周期，结构体其实也有这个概念，只不过我们之前对结构体的使用都停留在非引用类型字段上。

既然之前已经理解了生命周期，那么意味着在结构体中使用引用也变得可能：只要为结构体中的**每一个引用标注上生命周期**即可：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

结构体的生命周期标注语法跟泛型参数语法很像，需要对生命周期参数进行声明 `<'a>`。该生命周期标注说明，**结构体 `ImportantExcerpt` 所引用的字符串 `str` 必须比该结构体活得更久**。

下面的代码就无法通过编译：

```rust
#[derive(Debug)]
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let i;
    {
        let novel = String::from("Call me Ishmael. Some years ago...");
        let first_sentence = novel.split('.').next().expect("Could not find a '.'");
        i = ImportantExcerpt {
            part: first_sentence,
        };
    }
    println!("{:?}",i);
}
```

编译报错：

```rust
error[E0597]: `novel` does not live long enough
  --> src/main.rs:10:30
   |
10 |         let first_sentence = novel.split('.').next().expect("Could not find a '.'");
   |                              ^^^^^^^^^^^^^^^^ borrowed value does not live long enough
...
14 |     }
   |     - `novel` dropped here while still borrowed
15 |     println!("{:?}",i);
   |                     - borrow later used here
```



### 生命周期消除

实际上，对于编译器来说，每一个引用类型都有一个生命周期，那么为什么我们在使用过程中，很多时候无需标注生命周期？例如：

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

该函数的参数和返回值都是引用类型，尽管我们没有显式的为其标注生命周期，编译依然可以通过。其实原因不复杂，**编译器为了简化用户的使用，运用了生命周期消除大法**。在这种情况下编译器能够自动推导出，返回值的引用生命周期一定和参数`s`的生命周期相同，没有第二种情况，所以就可以省略生命周期参数。

有几点需要注意：

- 消除规则不是万能的，若编译器不能确定某件事是正确时，会直接判为不正确，那么还是需要手动标注生命周期
- **函数或者方法中，参数的生命周期被称为 `输入生命周期`，返回值的生命周期被称为 `输出生命周期`**

#### 三条消除规则

编译器使用三条消除规则来确定哪些场景不需要显式地去标注生命周期。其中第一条规则应用在输入生命周期上，第二、三条应用在输出生命周期上。若编译器发现三条规则都不适用时，就会报错，提示你需要手动标注生命周期。

1. **每一个引用参数都会获得独自的生命周期**

   例如一个引用参数的函数就有一个生命周期标注: `fn foo<'a>(x: &'a i32)`，两个引用参数的有两个生命周期标注:`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`, 依此类推。

2. **若只有一个输入生命周期(函数参数中只有一个引用类型)，那么该生命周期会被赋给所有的输出生命周期**，也就是所有返回值的生命周期都等于该输入生命周期

   例如函数 `fn foo(x: &i32) -> &i32`，`x` 参数的生命周期会被自动赋给返回值 `&i32`，因此该函数等同于 `fn foo<'a>(x: &'a i32) -> &'a i32`

3. **若存在多个输入生命周期，且其中一个是 `&self` 或 `&mut self`，则 `&self` 的生命周期被赋给所有的输出生命周期**

   拥有 `&self` 形式的参数，说明该函数是一个 `方法`，该规则让方法的使用便利度大幅提升。

若一个方法，它的返回值的生命周期就是跟参数 `&self` 的不一样怎么办？答案很简单：手动标注生命周期，因为这些规则只是编译器发现你没有标注生命周期时默认去使用的，当你标注生命周期后，编译器自然会乖乖听你的话。



### 方法中的生命周期

先来回忆下泛型的语法：

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

实际上，为具有生命周期的结构体实现方法时，我们使用的语法跟泛型参数语法很相似：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

其中有几点需要注意的：

- `impl` 中必须使用结构体的完整名称，包括 `<'a>`，因为*生命周期标注也是结构体类型的一部分*！
- 方法签名中，往往不需要标注生命周期，得益于生命周期消除的第一和第三规则

下面的例子展示了第三规则应用的场景：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

首先，编译器应用第一规则，给予每个输入参数一个生命周期:

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

接着，编译器应用第三规则，将 `&self` 的生命周期赋给返回值 `&str`：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'a str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

最开始的代码，尽管我们没有给方法标注生命周期，但是在第一和第三规则的配合下，编译器依然完美的为我们亮起了绿灯。

在结束这块儿内容之前，再来做一个有趣的修改，将方法返回的生命周期改为`'b`：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

此时，编译器会报错，因为编译器无法知道 `'a` 和 `'b` 的关系。 `&self` 生命周期是 `'a`，那么 `self.part` 的生命周期也是 `'a`，但是好巧不巧的是，我们手动为返回值 `self.part` 标注了生命周期 `'b`，因此编译器需要知道 `'a` 和 `'b` 的关系。

有一点很容易推理出来：由于 `&'a self` 是被引用的一方，因此引用它的 `&'b str` 必须要活得比它短，否则会出现悬垂引用。因此说明生命周期 `'b` 必须要比 `'a` 小，只要满足了这一点，编译器就不会再报错：

```rust
impl<'a: 'b, 'b> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

关键点：

- `'a: 'b`，是生命周期约束语法，跟泛型约束非常相似，用于说明 `'a` 必须比 `'b` 活得久
- 可以把 `'a` 和 `'b` 都在同一个地方声明（如上），或者分开声明但通过 `where 'a: 'b` 约束生命周期关系，如下：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'b str
    where
        'a: 'b,
    {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

总之，实现方法比想象中简单：加一个约束，就能暗示编译器，尽管引用吧，反正我想引用的内容比我活得久，爱咋咋地，我怎么都不会引用到无效的内容！



### 静态生命周期

在 Rust 中有一个非常特殊的生命周期，那就是 `'static`，拥有该生命周期的引用可以和整个程序活得一样久。

在之前我们学过字符串字面量，提到过它是被硬编码进 Rust 的二进制文件中，因此这些字符串变量全部具有 `'static` 的生命周期：

```rust
let s: &'static str = "我没啥优点，就是活得久，嘿嘿";
```

当生命周期不知道怎么标时，对类型施加一个静态生命周期的约束 `T: 'static` 是不是很爽？这样我和编译器再也不用操心它到底活多久了。

嗯，只能说，这个想法是对的，在不少情况下，`'static` 约束确实可以解决生命周期编译不通过的问题，但是问题来了：本来该引用没有活那么久，但是你非要说它活那么久，万一引入了潜在的 BUG 怎么办？遇到因为生命周期导致的编译不通过问题，首先想的应该是：是否是我们试图创建一个悬垂引用，或者是试图匹配不一致的生命周期，而不是简单粗暴的用 `'static` 来解决问题。

有时候，`'static` 确实可以帮助我们解决非常复杂的生命周期问题甚至是无法被手动解决的生命周期问题，那么此时就应该放心大胆的用，只要你确定：**你的所有引用的生命周期都是正确的，只是编译器太笨不懂罢了**。

总结下：

- 生命周期 `'static` 意味着能和程序活得一样久，例如字符串字面量和特征对象
- 实在遇到解决不了的生命周期标注问题，可以尝试 `T: 'static`，有时候它会给你奇迹



### 一个复杂的例子

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```



## 深入生命周期

### 不太聪明的生命周期检查

**例子1**

```rust
#[derive(Debug)]
struct Foo;

impl Foo {
    fn mutate_and_share(&mut self) -> &Self {
        &*self
    }
    fn share(&self) {}
}

fn main() {
    let mut foo = Foo;
    let loan = foo.mutate_and_share();
    foo.share();
    println!("{:?}", loan);
}
```

上面的代码中，`foo.mutate_and_share()` 虽然借用了 `&mut self`，但是它最终返回的是一个 `&self`，然后赋值给 `loan`，因此理论上来说它最终是进行了不可变借用，同时 `foo.share` 也进行了不可变借用，那么根据 Rust 的借用规则：多个不可变借用可以同时存在，因此该代码应该编译通过。

事实上，运行代码后，将看到错误：

```rust
error[E0502]: cannot borrow `foo` as immutable because it is also borrowed as mutable
  --> src/main.rs:12:5
   |
11 |     let loan = foo.mutate_and_share();
   |                ---------------------- mutable borrow occurs here
12 |     foo.share();
   |     ^^^^^^^^^^^ immutable borrow occurs here
13 |     println!("{:?}", loan);
   |                      ---- mutable borrow later used here
```

编译器的提示在这里其实有些难以理解，因为可变借用仅在 `mutate_and_share` 方法内部有效，出了该方法后，就只有返回的不可变借用，因此，按理来说可变借用不应该在 `main` 的作用范围内存在。

对于这个反直觉的事情，让我们用生命周期来解释下，可能就很好理解了：

```rust
struct Foo;

impl Foo {
    fn mutate_and_share<'a>(&'a mut self) -> &'a Self {
        &'a *self
    }
    fn share<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo: Foo = Foo;
        'c: {
            let loan: &'c Foo = Foo::mutate_and_share::<'c>(&'c mut foo);
            'd: {
                Foo::share::<'d>(&'d foo);
            }
            println!("{:?}", loan);
        }
    }
}
```

这就解释了可变借用为啥会在 `main` 函数作用域内有效，最终导致 `foo.share()` 无法再进行不可变借用。

总结下：`&mut self` 借用的生命周期和 `loan` 的生命周期相同，将持续到 `println` 结束。而在此期间 `foo.share()` 又进行了一次不可变 `&foo` 借用，违背了可变借用与不可变借用不能同时存在的规则，最终导致了编译错误。

上述代码实际上完全是正确的，但目前遇到这种情况，只能修改代码去满足编译器的要求，并期待以后它会更聪明。

**例子2**

```rust
#![allow(unused)]
fn main() {
    use std::collections::HashMap;
    use std::hash::Hash;
    fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
    where
        K: Clone + Eq + Hash,
        V: Default,
    {
        match map.get_mut(&key) {
            Some(value) => value,
            None => {
                map.insert(key.clone(), V::default());
                map.get_mut(&key).unwrap()
            }
        }
    }
}
```

这段代码不能通过编译的原因是编译器未能精确地判断出某个可变借用不再需要，反而谨慎的给该借用安排了一个很大的作用域，结果导致后续的借用失败：

```rust
error[E0499]: cannot borrow `*map` as mutable more than once at a time
  --> src/main.rs:13:17
   |
5  |       fn get_default<'m, K, V>(map: &'m mut HashMap<K, V>, key: K) -> &'m mut V
   |                      -- lifetime `'m` defined here
...
10 |           match map.get_mut(&key) {
   |           -     ----------------- first mutable borrow occurs here
   |  _________|
   | |
11 | |             Some(value) => value,
12 | |             None => {
13 | |                 map.insert(key.clone(), V::default());
   | |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ second mutable borrow occurs here
14 | |                 map.get_mut(&key).unwrap()
15 | |             }
16 | |         }
   | |_________- returning this value requires that `*map` is borrowed for `'m`
```

分析代码可知在 `match map.get_mut(&key)` 方法调用完成后，对 `map` 的可变借用就可以结束了。但从报错看来，编译器不太聪明，它认为该借用会持续到整个 `match` 语句块的结束(第 16 行处)，这便造成了后续借用的失败。



### 无界生命周期

不安全代码(`unsafe`)经常会凭空产生引用或生命周期，这些生命周期被称为是 **无界(unbound)** 的。

无界生命周期往往是在解引用一个裸指针(裸指针 raw pointer)时产生的，换句话说，它是凭空产生的，因为输入参数根本就没有这个生命周期：

```rust
fn f<'a, T>(x: *const T) -> &'a T {
    unsafe {
        &*x
    }
}
```

上述代码中，参数 `x` 是一个裸指针，它并没有任何生命周期，然后通过 `unsafe` 操作后，它被进行了解引用，变成了一个 Rust 的标准引用类型，该类型必须要有生命周期，也就是 `'a`。

可以看出 `'a` 是凭空产生的，因此它是无界生命周期。这种生命周期由于没有受到任何约束，因此它想要多大就多大。

我们在实际应用中，要尽量避免这种无界生命周期。最简单的避免无界生命周期的方式就是在函数声明中运用生命周期消除规则。**若一个输出生命周期被消除了，那么必定因为有一个输入生命周期与之对应**。



### 生命周期约束 HRTB

生命周期约束跟特征约束类似，都是通过形如 `'a: 'b` 的语法，来说明两个生命周期的长短关系。

#### \'a: \'b

假设有两个引用 `&'a i32` 和 `&'b i32`，它们的生命周期分别是 `'a` 和 `'b`，若 `'a` >= `'b`，则可以定义 `'a:'b`，表示 `'a` 至少要活得跟 `'b` 一样久。

```rust
struct DoubleRef<'a,'b:'a, T> {
    r: &'a T,
    s: &'b T
}
```

由于我们使用了生命周期约束 `'b: 'a`，因此 `'b` 必须活得比 `'a` 久，也就是结构体中的 `s` 字段引用的值必须要比 `r` 字段引用的值活得要久。

#### T: \'a

表示类型 `T` 必须比 `'a` 活得要久：

```rust
struct Ref<'a, T: 'a> {
    r: &'a T
}
```

因为结构体字段 `r` 引用了 `T`，因此 `r` 的生命周期 `'a` 必须要比 `T` 的生命周期更短(被引用者的生命周期必须要比引用长)。

在 Rust 1.30 版本之前，该写法是必须的，但是从 1.31 版本开始，编译器可以自动推导 `T: 'a` 类型的约束，因此我们只需这样写即可：

```rust
struct Ref<'a, T> {
    r: &'a T
}
```

来看一个使用了生命周期约束的综合例子：

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a: 'b, 'b> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

上面的例子中必须添加约束 `'a: 'b` 后，才能成功编译，因为 `self.part` 的生命周期与 `self`的生命周期一致，将 `&'a` 类型的生命周期强行转换为 `&'b` 类型，会报错，只有在 `'a` >= `'b` 的情况下，`'a` 才能转换成 `'b`。



### 闭包函数的消除规则

一段简单的代码：

```rust
fn main() {
    fn fn_elision(x: &i32) -> &i32 {
        x
    }
    let closure_slision = |x: &i32| -> &i32 { &x };
}
```

乍一看，这段代码平平无奇，能有什么问题呢？一运行：

```rust
error: lifetime may not live long enough
 --> src/main.rs:7:47
  |
7 |     let closure_slision = |x: &i32| -> &i32 { &x };
  |                               -        -      ^^ returning this value requires that `'1` must outlive `'2`
  |                               |        |
  |                               |        let's call the lifetime of this reference `'2`
  |                               let's call the lifetime of this reference `'1`
```

错误原因是编译器无法推测返回的引用和传入的引用谁活得更久！

真的是非常奇怪的错误，之前学过这样一条生命周期消除规则：**如果函数参数中只有一个引用类型，那该引用的生命周期会被自动分配给所有的返回引用**。我们当前的情况完美符合， `function` 函数的顺利编译通过，就充分说明了问题。

那为什么会报错呢？ 如果我们把闭包函数稍微改一下，就能通过了：

```rust
fn main() {
    let m = 1;
    fn fn_elision(x: &i32) -> &i32 {
        x
    }
    let closure_slision = |x: &i32| -> &i32 { &m };
}
```

如果不返回`&x`，而是返回`&m`，这样就能顺利编译。

其实`fn_elision`以及返回`&m`都能正常编译，原因在于这种情况下，生命周期都是确定性的：普通函数只有一个引用参数和一个引用返回值，那么只有一种可能，返回值来自输入参数并且其生命周期相同。闭包中返回`&m`也是确定性的，返回值生命周期等于`m`变量的生命周期。
但是返回值是`&x`时，由于是闭包函数，情况就复杂了，因为闭包函数其上下文非常复杂，有可能只是像普通函数一样调用，也有可能作为一个迭代函数的回调函数调用，如果作为回调函数调用，其生命周期就完全无法预知了，很可能会出现迭代函数调用回调后，将回调函数的值返回出来，但是调用回调函数的输入参数却已经被释放了。Rust作为内存安全的语言，决不能让这种事情发生，所以保守起见，这种闭包函数定义无法通过编译。

因此：**这个问题，可能无法解决，还是老老实实用正常的函数，不要秀闭包了**。



### NLL (Non-Lexical Lifetime)

之前在引用与借用那一章有讲到过这个概念，简单来说就是：**引用的生命周期正常来说应该从借用开始一直持续到作用域结束**，但是这种规则会让多引用共存的情况变得更复杂：

```rust
fn main() {
   let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // 新编译器中，r1,r2作用域在这里结束

    let r3 = &mut s;
    println!("{}", r3);
}
```

好在，该规则从 1.31 版本引入 `NLL` 后，就变成了：**引用的生命周期从借用处开始，一直持续到最后一次使用的地方**。

按照最新的规则，我们再来分析一下上面的代码。`r1` 和 `r2` 不可变借用在 `println!` 后就不再使用，因此生命周期也随之结束，那么 `r3` 的借用就不再违反借用的规则，皆大欢喜。

#### Reborrow 再借用

学完 `NLL` 后，我们就有了一定的基础，可以继续学习关于借用和生命周期的一个高级内容：**再借用**。

看一段代码：

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn move_to(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
    }
}

fn main() {
    let mut p = Point { x: 0, y: 0 };
    let r = &mut p;
    let rr: &Point = &*r;

    println!("{:?}", rr);
    r.move_to(10, 10);
    println!("{:?}", r);
}
```

以上代码，可能会觉得可变引用 `r` 和不可变引用 `rr` 同时存在会报错吧？但是事实上并不会，原因在于 `rr` 是对 `r` 的再借用。

对于再借用而言，`rr` 再借用时不会破坏借用规则，但是你不能在它的生命周期内再使用原来的借用 `r`，来看看对上段代码的分析：

```rust
fn main() {
    let mut p = Point { x: 0, y: 0 };
    let r = &mut p;
    // reborrow! 此时对`r`的再借用不会导致跟上面的借用冲突
    let rr: &Point = &*r;

    // 再借用`rr`最后一次使用发生在这里，在它的生命周期中，我们并没有使用原来的借用`r`，因此不会报错
    println!("{:?}", rr);

    // 再借用结束后，才去使用原来的借用`r`
    r.move_to(10, 10);
    println!("{:?}", r);
}
```

再来看一个例子：

```rust
use std::vec::Vec;
fn read_length(strings: &mut Vec<String>) -> usize {
   strings.len()
}
```

如上所示，函数体内对参数的二次借用也是典型的 `Reborrow` 场景。



### 生命周期消除规则补充

在上一节中，我们介绍了三大基础生命周期消除规则，实际上，随着 Rust 的版本进化，该规则也在不断演进，这里再介绍几个常见的消除规则：

#### impl 块消除

```rust
impl<'a> Reader for BufReader<'a> {
    // methods go here
    // impl内部实际上没有用到'a
}
```

如果你以前写的`impl`块长上面这样，同时在 `impl` 内部的方法中，根本就没有用到 `'a`，那就可以写成下面的代码形式。

```rust
impl Reader for BufReader<'_> {
    // methods go here
}
```

`'_` 生命周期表示 `BufReader` 有一个不使用的生命周期，我们可以忽略它，无需为它创建一个名称。

有个疑问：既然用不到 `'a`，为何还要写出来？如果仔细回忆下上一节的内容，里面有一句专门用粗体标注的文字：**生命周期参数也是类型的一部分**，因此 `BufReader<'a>` 是一个完整的类型，在实现它的时候，你不能把 `'a` 给丢了！

#### 生命周期约束消除

```rust
// Rust 2015
struct Ref<'a, T: 'a> {
    field: &'a T
}

// Rust 2018
struct Ref<'a, T> {
    field: &'a T
}
```

新版本 Rust 中，上面情况中的 `T: 'a` 可以被消除掉，当然，你也可以显式的声明，但是会影响代码可读性。关于类似的场景，Rust 团队计划在未来提供更多的消除规则，但是，你懂的，计划未来就等于未知。



### 一个复杂的例子

下面是一个关于生命周期声明过大的例子，会较为复杂，细细阅读，它能帮我们对生命周期的理解更加深入。

```rust
struct Interface<'a> {
    manager: &'a mut Manager<'a>
}

impl<'a> Interface<'a> {
    pub fn noop(self) {
        println!("interface consumed");
    }
}

struct Manager<'a> {
    text: &'a str
}

struct List<'a> {
    manager: Manager<'a>,
}

impl<'a> List<'a> {
    pub fn get_interface(&'a mut self) -> Interface {
        Interface {
            manager: &mut self.manager
        }
    }
}

fn main() {
    let mut list = List {
        manager: Manager {
            text: "hello"
        }
    };

    list.get_interface().noop();

    println!("Interface should be dropped here and the borrow released");

    // 下面的调用会失败，因为同时有不可变/可变借用
    // 但是Interface在之前调用完成后就应该被释放了
    use_list(&list);
}

fn use_list(list: &List) {
    println!("{}", list.manager.text);
}
```

这段代码看上去并不复杂，实际上难度挺高的，首先在直觉上，`list.get_interface()` 借用的可变引用，按理来说应该在这行代码结束后，就归还了，但是为什么还能持续到 `use_list(&list)` 后面呢？

这是因为我们在 `get_interface` 方法中声明的 `lifetime` 有问题，该方法的参数的生命周期是 `'a`，而 `List` 的生命周期也是 `'a`，说明该方法至少活得跟 `List` 一样久，再回到 `main` 函数中，`list` 可以活到 `main` 函数的结束，因此 `list.get_interface()` 借用的可变引用也会被认为活到 `main` 函数的结束，在此期间，自然无法再进行借用了。

要解决这个问题，我们需要为 `get_interface` 方法的参数给予一个不同于 `List<'a>` 的生命周期 `'b`，最终代码如下：

```rust
struct Interface<'b, 'a: 'b> {
    manager: &'b mut Manager<'a>
}

impl<'b, 'a: 'b> Interface<'b, 'a> {
    pub fn noop(self) {
        println!("interface consumed");
    }
}

struct Manager<'a> {
    text: &'a str
}

struct List<'a> {
    manager: Manager<'a>,
}

impl<'a> List<'a> {
    pub fn get_interface<'b>(&'b mut self) -> Interface<'b, 'a>
    where 'a: 'b {
        Interface {
            manager: &mut self.manager
        }
    }
}

fn main() {

    let mut list = List {
        manager: Manager {
            text: "hello"
        }
    };

    list.get_interface().noop();

    println!("Interface should be dropped here and the borrow released");

    // 下面的调用可以通过，因为Interface的生命周期不需要跟list一样长
    use_list(&list);
}

fn use_list(list: &List) {
    println!("{}", list.manager.text);
}
```



## \'static 和 T: \'static

两个小例子：

```rust
fn main() {
	let mark_twain: &str = "Samuel Clemens"; // 字符串字面值就具有 'static 生命周期
	print_author(mark_twain);
}
fn print_author(author: &'static str) {
    println!("{}", author);
}
```

```rust
use std::fmt::Display;
fn main() {
    let mark_twain = "Samuel Clemens";
    print(&mark_twain);
}

fn print<T: Display + 'static>(message: &T) {  // 'static 作为生命周期约束来使用
    println!("{}", message);
}
```

### \'static

`&'static` 对生命周期有非常强的要求：一个引用必须要活得跟整个程序一样久，才能被标注为 `&'static`。

对于字符串字面量来说，它直接被打包到二进制文件中，永远不会被 `drop`，因此它能跟程序活得一样久，自然它的生命周期是 `'static`。

但是，**`&'static` 生命周期针对的仅仅是引用，而不是持有该引用的变量，对于变量来说，还是要遵循相应的作用域规则** :

```rust
use std::{slice::from_raw_parts, str::from_utf8_unchecked};

fn get_memory_location() -> (usize, usize) {
    // “Hello World” 是字符串字面量，因此它的生命周期是 `'static`.
    // 但持有它的变量 `string` 的生命周期就不一样了，它完全取决于变量作用域，对于该例子来说，也就是当前的函数范围
    let string = "Hello World!";
    let pointer = string.as_ptr() as usize;
    let length = string.len();
    (pointer, length)
    // `string` 在这里被 drop 释放
    // 虽然变量被释放，无法再被访问，但是数据依然还会继续存活
}

fn get_str_at_location(pointer: usize, length: usize) -> &'static str {
  	// 使用裸指针需要 `unsafe{}` 语句块
  	unsafe { from_utf8_unchecked(from_raw_parts(pointer as *const u8, length)) }
}

fn main() {
    let (pointer, length) = get_memory_location();
    let message = get_str_at_location(pointer, length);
    println!(
        "The {} bytes at 0x{:X} stored: {}",
        length, pointer, message
    );
    // 如果大家想知道为何处理裸指针需要 `unsafe`，可以试着反注释以下代码
    // let message = get_str_at_location(1000, 10);
}

```

上面代码有两点值得注意：

- `&'static` 的引用确实可以和程序活得一样久，因为我们通过 `get_str_at_location` 函数直接取到了对应的字符串
- 持有 `&'static` 引用的变量，它的生命周期受到作用域的限制，务必不要搞混了

### T: \'static

```rust
use std::fmt::Debug;

fn print_it<T: Debug + 'static>( input: T) {
    println!( "'static value passed in is: {:?}", input );
}

fn print_it1( input: impl Debug + 'static ) {
    println!( "'static value passed in is: {:?}", input );
}



fn main() {
    let i = 5;

    print_it(&i);
    print_it1(&i);
}
```

以上代码会报错，原因很简单: `&i` 的生命周期无法满足 `'static` 的约束，如果大家将 `i` 修改为常量，那自然一切 OK。

另外如下修改也能编译通过：

```rust
use std::fmt::Debug;

fn print_it<T: Debug + 'static>( input: &T) {
    println!( "'static value passed in is: {:?}", input );
}

fn main() {
    let i = 5;

    print_it(&i);
}
```

这段代码竟然不报错了！原因在于我们约束的是 `T`，但是使用的却是它的引用 `&T`，换而言之，我们根本没有直接使用 `T`，因此编译器就没有去检查 `T` 的生命周期约束！它只要确保 `&T` 的生命周期符合规则即可，在上面代码中，它自然是符合的。

再看一个例子：

```rust
use std::fmt::Display;

fn main() {
    let r1;
    let r2;
    {
        static STATIC_EXAMPLE: i32 = 42;
        r1 = &STATIC_EXAMPLE;
        let x = "&'static str";
        r2 = x;
        // r1 和 r2 持有的数据都是 'static 的，因此在花括号结束后，并不会被释放
    }

    println!("&'static i32: {}", r1); // -> 42
    println!("&'static str: {}", r2); // -> &'static str

    let r3: &str;

    {
        let s1 = "String".to_string();

        // s1 虽然没有 'static 生命周期，但是它依然可以满足 T: 'static 的约束
        // 充分说明这个约束是多么的弱。。
        static_bound(&s1);

        // s1 是 String 类型，没有 'static 的生命周期，因此下面代码会报错
        r3 = &s1;

        // s1 在这里被 drop
    }
    println!("{}", r3);
}

fn static_bound<T: Display + 'static>(t: &T) {
	println!("{}", t);
}
```



### static到底针对谁

到底是针对 `&'static` 这个引用还是该引用指向的数据活得跟程序一样久呢？

**答案是引用指向的数据**，而引用本身是要遵循其作用域范围的，我们来简单验证下：

```rust
fn main() {
    {
        let static_string = "I'm in read-only memory";
        println!("static_string: {}", static_string);

        // 当 `static_string` 超出作用域时，该引用不能再被使用，但是数据依然会存在于 binary 所占用的内存中
    }

    println!("static_string reference remains alive: {}", static_string);
}
```

以上代码不出所料会报错，原因在于虽然字符串字面量 "I'm in read-only memory" 的生命周期是 `'static`，但是持有它的引用并不是，它的作用域在内部花括号 `}` 处就结束了。

## 总结

总之， `&'static` 和 `T: 'static` 大体上相似，相比起来，后者的使用形式会更加复杂一些。

至此，对于 `'static` 和 `T: 'static` 也有了清晰的理解，那么我们应该如何使用它们呢？

作为经验之谈，可以这么来:

- 如果你需要添加 `&'static` 来让代码工作，那很可能是设计上出问题了
- 如果你希望满足和取悦编译器，那就使用 `T: 'static`，很多时候它都能解决问题
