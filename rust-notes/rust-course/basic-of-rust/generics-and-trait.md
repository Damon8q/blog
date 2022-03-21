# 泛型(generics) 和 特征(trait)

泛型和特征是 Rust 中最最重要的抽象类型，也是学习 Rust 路上的拦路虎，但是挑战往往与乐趣并存，一旦学会，在后面学习 Rust 的路上，将一往无前。

# 泛型 Generics

编程中经常有这样的需求：用同一功能的函数处理不同类型的数据，例如两个数的加法，无论是整数还是浮点数，甚至是自定义类型，都能支持。如果不支持泛型，通常需要为每种类型编写一个函数：

```rust
fn add_i8(a:i8, b:i8) -> i8 {
    a + b
}
fn add_i32(a:i32, b:i32) -> i32 {
    a + b
}
fn add_f64(a:f64, b:f64) -> f64 {
    a + b
}

fn main() {
    println!("add i8: {}", add_i8(2i8, 3i8));
    println!("add i32: {}", add_i32(20, 30));
    println!("add f64: {}", add_f64(1.23, 1.23));
}
```

这样的代码会很啰嗦。实际上，泛型就是一种多态。泛型主要目的是为程序员提供编程的便利，减少代码的臃肿。写法大致如下：

```rust
fn add<T>(a:T, b:T) -> T {
    a + b
}

fn main() {
    println!("add i8: {}", add(2i8, 3i8));
    println!("add i32: {}", add(20, 30));
    println!("add f64: {}", add(1.23, 1.23));
}
```

这段代码虽然很简洁，但是并不能编译通过，在后面进行详细讲解。

## 泛型详解

上面代码的 `T` 就是**泛型参数**，泛型参数的名称你可以任意起，但是出于惯例，都用 `T` ( `T` 是 `type` 的首字母)来作为首选，这个名称越短越好，除非需要表达含义，否则一个字母是最完美的。

使用泛型参数，有一个先决条件，必需在使用前对其进行声明：

```rust
fn largest<T>(list: &[T]) -> T {
```

该泛型函数的作用是从列表中找出最大的值，其中列表中的元素类型为 T。首先 `largest<T>` 对泛型参数 `T` 进行了声明，然后才在函数参数中进行使用该泛型参数 `list: &[T]` （还记得 `&[T]` 类型吧？这是**数组切片**）。

可以这样理解这个函数：函数 `largest` 有泛型类型 `T`，它有个参数 `list`，其类型是元素为 `T` 的数组切片，最后，该函数返回值的类型也是 `T`。

具体实现：

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];                    
    // 这里的`&item`可以写成`item`吗？
    // 可以。因为iter迭代器返回的类型是`&T`
    // `&item`代表是会发生所有权转移的，如果需要实现Copy trait
    // `item`则获取的是引用，不会发生所有权转移
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }

    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

运行后报错：

```rust
error[E0369]: binary operation `>` cannot be applied to type `T` // `>`操作符不能用于类型`T`
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T` // 考虑对T进行类型上的限制 :
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ++++++++++++++++++++++
```

因为 `T` 可以是任何类型，但不是所有的类型都能进行比较，因此上面的错误中，编译器建议我们给 `T` 添加一个类型限制：使用 `std::cmp::PartialOrd` 特征（Trait）对 `T` 进行限制，特征在后续章节会介绍，这里只需要理解该特征的目的就是让**类型实现可比较的功能**。

还有一开始的 `add` 泛型函数，如果你运行它，会得到以下的报错：

```rust
error[E0369]: cannot add `T` to `T` // 无法将 `T` 类型跟 `T` 类型进行相加
 --> src/main.rs:2:7
  |
2 |     a + b
  |     - ^ - T
  |     |
  |     T
  |
help: consider restricting type parameter `T`
  |
1 | fn add<T: std::ops::Add<Output = T>>(a:T, b:T) -> T {
  |         +++++++++++++++++++++++++++
```

同样的，不是所有 `T` 类型都能进行相加操作，因此我们需要用 `std::ops::Add<Output = T>` 对 `T` 进行限制：

```rust
fn add<T: std::ops::Add<Output = T>>(a:T, b:T) -> T {
    a + b
}
```

进行如上修改后，就可以正常运行。

## 结构体中使用泛型

结构体中的字段类型也可以用泛型来定义，下面代码定义了一个坐标点 `Point`，它可以存放任何类型的坐标值：

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point{ x: 5, y: 10};
    let float = Point{x: 1.0, y: 4.0};    
}
```

有两点需要特别注意：

* **提前声明**。跟泛型函数定义类似，首先我们在使用泛型参数之前必需要进行声明 `Point<T>`，接着就可以在结构体的字段类型中使用 `T` 来替代具体的类型

* **x和y是相同的类型**

如果想让 `x` 和 `y` 即能类型相同，又能类型不同，需要使用不同的泛型参数：

```rust
struct Point<T,U> {
    x: T,
    y: U,
}
fn main() {
    let p = Point{x: 1, y :1.1};
}
```

切记，所有的泛型参数都要提前声明：`Point<T,U>` ! 但是如果你的结构体变成这鬼样：`struct Woo<T,U,V,W,X>`，那么你需要考虑拆分这个结构体，减少泛型参数的个数和代码复杂度。

## 枚举中使用泛型

`Option` 是最常用的枚举类型：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

泛型的引入，我们可以在任何一个需要返回值的函数中，使用 `Option<T>` 枚举类型来做返回值，用于返回一个任意类型的值 `Some(T)`，或者没有值 `None`。

还有另外一个常用的枚举类型： `Result`：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

这个枚举和 `Option` 一样，主要用于函数返回值，与 `Option` 用于值的存在与否不同，`Result` 关注的主要是值的正确性。

如果函数正常运行，则最后返回一个 `Ok(T)`，`T` 是函数具体的返回值类型，如果函数异常运行，则返回一个 `Err(E)`，`E` 是错误类型。例如打开一个文件：如果成功打开文件，则返回 `Ok(std::fs::File)`，因此 `T` 对应的是 `std::fs::File` 类型；而当打开文件时出现问题时，返回 `Err(std::io::Error)`，`E` 对应的就是 `std::io::Error` 类型。

## 方法中使用泛型

方法上也可以使用泛型：

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

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

使用泛型参数前，依然需要提前声明：`impl<T>`，只有提前声明了，我们才能在`Point<T>`中使用它，这样 Rust 就知道 `Point` 的尖括号中的类型是泛型而不是具体类型。

需要注意的是，这里的 `Point<T>` 不再是泛型声明，而是一个完整的结构体类型，因为我们定义的结构体就是 `Point<T>` 而不再是 `Point`。

除了结构体中的泛型参数，我们还能在该结构体的方法中定义额外的泛型参数，就跟泛型函数一样：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c'};

    let p3 = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

这个例子中，`T,U` 是定义在结构体 `Point` 上的泛型参数，`V,W` 是单独定义在方法 `mixup` 上的泛型参数，它们并不冲突，可以理解为一个是结构体泛型，一个是函数泛型。

### 为具体的泛型类型实现方法

对于 `Point<T>` 类型，你不仅能定义基于 `T` 的方法，还能针对特定的具体类型，进行方法定义：

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

这段代码意味着 `Point<f32>` 类型会有一个方法 `distance_from_origin`，而其他 `T` 不是 `f32` 类型的 `Point<T>` 实例则没有定义此方法。这个方法计算点实例与坐标`(0.0, 0.0)` 之间的距离，并使用了只能用于浮点型的数学运算符。

这样我们就能针对特定的泛型类型实现某个特定的方法，对于其它泛型类型则没有定义该方法。

## const泛型

在之前的泛型中，可以抽象为一句话：针对类型实现的泛型，所有的泛型都是为了抽象不同的类型，那有没有针对值的泛型？从数组讲起。

在数组章节，有提到过一个重点：`[i32; 2]`和`[i32; 3]`是不同的数组类型，比如：

```rust
fn display_array(arr: [i32; 3]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(arr);

    let arr: [i32;2] = [1,2];
    display_array(arr);
}
```

运行后报错：

```rust
error[E0308]: mismatched types // 类型不匹配
  --> src/main.rs:10:19
   |
10 |     display_array(arr);
   |                   ^^^ expected an array with a fixed size of 3 elements, found one with 2 elements
                          // 期望一个长度为3的数组，却发现一个长度为2的
```

首先，让我们修改代码，让 `display_array` 能打印任意长度的 `i32` 数组：

```rust
fn display_array(arr: &[i32]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(&arr);

    let arr: [i32;2] = [1,2];
    display_array(&arr);
}
```

很简单，只要使用数组切片，然后传入 `arr` 的不可变引用即可。

接着，将 `i32` 改成所有类型的数组：

```rust
fn display_array<T: std::fmt::Debug>(arr: &[T]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(&arr);

    let arr: [i32;2] = [1,2];
    display_array(&arr);
}
```

也不难，唯一要注意的是需要对 `T` 加一个限制 `std::fmt::Debug`，该限制表明 `T` 可以用在 `println!("{:?}", arr)` 中，因为 `{:?}` 形式的格式化输出需要 `arr` 实现该trait。

通过引用，我们可以很轻松的解决处理任何类型数组的问题，但是如果在某些场景下引用不适宜用或者干脆不能用呢？好在，有了 const 泛型，也就是针对值的泛型，正好可以用于处理数组长度的问题：

```rust
fn display_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}
fn main() {
    let arr: [i32; 3] = [1, 2, 3];
    display_array(arr);

    let arr: [i32; 2] = [1, 2];
    display_array(arr);
}
```

如上所示，我们定义了一个类型为 `[T; N]` 的数组，其中 `T` 是一个基于类型的泛型参数，这个和之前讲的泛型没有区别，而重点在于 `N` 这个泛型参数，它是一个基于值的泛型参数！因为它用来替代的是数组的长度。

`N` 就是 const 泛型，定义的语法是 `const N: usize`，表示 const 泛型 `N` ，它基于的值类型是 `usize`。

在泛型参数之前，Rust 完全不适合复杂矩阵的运算，自从有了 const 泛型，一切即将改变。

### const泛型表达式

假设我们某段代码需要在内存很小的平台上工作，因此需要限制函数参数占用的内存大小，此时就可以使用 const 泛型表达式来实现：

```rust
// 目前只能在nightly版本下使用
#![allow(incomplete_features)]
#![feature(generic_const_exprs)]

fn something<T>(val: T)
where
    Assert<{ core::mem::size_of::<T>() < 768 }>: IsTrue,
    //       ^-----------------------------^ 这里是一个 const 表达式，换成其它的 const 表达式也可以
{
    //
}

fn main() {
    something([0u8; 0]); // ok
    something([0u8; 512]); // ok
    something([0u8; 1024]); // 编译错误，数组长度是1024字节，超过了768字节的参数长度限制
}

// ---

pub enum Assert<const CHECK: bool> {
    //
}

pub trait IsTrue {
    //
}

impl IsTrue for Assert<true> {
    //
}
```

## 泛型的性能

在 Rust 中泛型是零成本的抽象，意味着在使用泛型时，完全不用担心性能上的问题。

但是任何选择都是权衡得失的，既然我们获得了性能上的巨大优势，那么又失去了什么呢？Rust 是在编译期为泛型对应的多个类型，生成各自的代码，因此损失了编译速度和增大了最终生成文件的大小。

Rust 通过在编译时进行泛型代码的 **单态化**(*monomorphization*)来保证效率。单态化是一个通过填充编译时使用的具体类型，将通用代码转换为特定代码的过程。

编译器所做的工作正好与我们创建泛型函数的步骤相反，编译器寻找所有泛型代码被调用的位置并针对具体类型生成代码。

看一个使用标准库中 `Option` 枚举的例子：

```rust
let integer = Some(5);
let float = Some(5.0);
```

编译器会读取传递给 `Option<T>` 的值并发现有两种 `Option<T>`：一种对应 `i32` 另一种对应 `f64`。为此，它会将泛型定义 `Option<T>` 展开为 `Option_i32` 和 `Option_f64`，接着将泛型定义替换为这两个具体的定义。

编译器生成的单态化版本的代码看起来像这样：

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

# 特征 Trait

特征定义了可以被共享的行为，只要实现了特征，就能使用该行为。

## 定义特征

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

这里使用 `trait` 关键字来声明一个特征，`Summary` 是特征名。在大括号中定义了该特征的所有方法，在这个例子中是： `fn summarize(&self) -> String`。

特征只定义行为看起来是什么样，而不定义行为具体行为。因此，通常只定义特征方法的签名，而不进行实现，此时方法签名结尾是 `;`，而不是一个 `{}`。

## 为类型实现特征

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
pub struct Post {
    pub title: String, // 标题
    pub author: String, // 作者
    pub content: String, // 内容
}

impl Summary for Post {
    fn summarize(&self) -> String {
        format!("文章{}, 作者是{}", self.title, self.author)
    }
}

pub struct Weibo {
    pub username: String,
    pub content: String
}

impl Summary for Weibo {
    fn summarize(&self) -> String {
        format!("{}发表了微博{}", self.username, self.content)
    }
}
```

接下来就可以在这个类型上调用特征的方法：

```rust
fn main() {
    let post = Post{title: "Rust语言简介".to_string(),author: "Sunface".to_string(), content: "Rust棒极了!".to_string()};
    let weibo = Weibo{username: "sunface".to_string(),content: "好像微博没Tweet好用".to_string()};

    println!("{}",post.summarize());
    println!("{}",weibo.summarize());
}
```

### 特征定义与实现的位置（孤儿规则）

上面将 `Summary` 定义成了 `pub` 公开的。这样，如果他人想要使用`Summary` 特征，则可以引入到他们的包中，然后再进行实现。

重要的原则：**如果想要为类型 `A` 实现特征 `T`，那么 `A` 或者 `T` 至少有一个是在当前作用域中定义的！**。

例如可以为上面的 `Post` 类型实现标准库中的 `Display` 特征，这是因为 `Post` 类型定义在当前的作用域中。同时，我们也可以在当前包中为 `String` 类型实现 `Summary` 特征，因为 `Summary` 定义在当前作用域中。

但是你无法在当前作用域中，为 `String` 类型实现 `Display` 特征，因为它们俩都定义在标准库中，其定义所在的位置都不在当前作用域，跟你半毛钱关系都没有，看看就行了。

该规则被称为**孤儿规则**，可以确保其它人编写的代码不会破坏你的代码，也确保了你不会莫名其妙就破坏了风马牛不相及的代码。

### 默认实现

可以在特征中定义具有**默认实现**的方法，这样其它类型无需再实现该方法，或者也可以选择重载该方法：

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

默认实现允许调用相同特征中的其他方法，哪怕这些方法没有默认实现。例如：

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

为了使用 `Summary`，只需要实现 `summarize_author` 方法即可。

## 使用特征作为函数参数

定义一个函数，使用特征用做函数参数：

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

`impl Summary`，故名思义，它的意思是 `实现了Summary特征` 的 `item` 参数。

可以使用任何实现了 `Summary` 特征的类型作为该函数的参数，同时在函数体内，还可以调用该特征的方法，例如 `summarize` 方法。

## 特征约束(trait bound)

虽然 `impl Trait` 这种语法非常好理解，但是实际上它只是一个语法糖：

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

真正的完整书写形式如上所述，形如 `T: Summary` 被称为**特征约束**。

在简单的场景下 `impl Trait` 的语法就足够使用，但是对于复杂的场景，特征约束可以让我们拥有更大的灵活性和语法表现能力，例如一个函数接受两个 `impl Summary` 的参数：

```rust
pub fn notify(item1: &impl Summary, item2: &impl Summary) {}
```

如果我们想要强制函数的两个参数是同一类型呢？上面的语法就无法做到这种限制，此时我们只能使特征约束来实现：

```rust
pub fn notify<T: Summary>(item1: &T, item2: &T) {}
```

泛型类型 `T` 说明了 `item1` 和 `item2` 必须拥有同样的类型，同时 `T: Summary` 说明了 `T` 必须实现 `Summary` 特征。

### 多重约束

除了单个约束条件，还可以指定多个约束条件，例如除了让参数实现 `Summary` 特征外，还可以让参数实现 `Display` 特征以控制它的格式化输出：

```rust
pub fn notify(item: &(impl Summary + Display)) {}
```

除了上述的语法糖形式，还能使用特征约束的形式：

```rust
pub fn notify<T: Summary + Display>(item: &T) {}
```

通过这两个特征，就可以使用 `item.summarize` 方法，以及通过 `println!("{}", item)` 来格式化输出 `item`。

### Where 约束

当特征约束变得很多时，函数的签名将变得很复杂：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {
```

通过`where`改进：

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

### 使用特征约束有条件地实现方法或特征

特征约束，可以让我们在指定类型 + 指定特征的条件下去实现方法，例如：

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self {
            x,
            y,
        }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

`cmd_display` 方法，并不是所有的 `Pair<T>` 结构体对象都可以拥有，只有 `T` 同时实现了 `Display + PartialOrd` 的 `Pair<T>` 才可以拥有此方法。

**也可以有条件地实现特征**, 例如，标准库为任何实现了 `Display` 特征的类型实现了 `ToString` 特征：

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

我们可以对任何实现了 `Display` 特征的类型调用由 `ToString` 定义的 `to_string` 方法。例如，可以将整型转换为对应的 `String` 值，因为整型实现了 `Display`：

```rust
let s = 3.to_string();   
```

## 函数返回中的 impl Trait

可以通过 `impl Trait` 来说明一个函数返回了一个类型，该类型实现了某个特征：

```rust
fn returns_summarizable() -> impl Summary {
    Weibo {
        username: String::from("sunface"),
        content: String::from(
            "m1 max太厉害了，电脑再也不会卡",
        )
    }
}
```

要注意的是，对于 `returns_summarizable` 的调用者而言，他只知道返回了一个实现了 `Summary` 特征的对象，但是并不知道返回了一个 `Weibo` 类型。

这种 `impl Trait` 形式的返回值，在一种场景下非常非常有用，那就是返回的真实类型非常复杂，你不知道该怎么声明时(毕竟 Rust 要求你必须标出所有的类型)，此时就可以用 `impl Trait` 的方式简单返回。

例如，闭包和迭代器就是很复杂，可以用 `impl Iterator` 来告诉调用者，返回了一个迭代器，因为所有迭代器都会实现 `Iterator` 特征。

但是这种返回值方式有一个很大的限制：只能有一个具体的类型，例如：

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        Post {
            title: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Weibo {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
        }
    }
}
```

以上的代码就无法通过编译，因为它返回了两个不同的类型 `Post` 和 `Weibo`。

```rust
`if` and `else` have incompatible types
expected struct `Post`, found struct `Weibo`
```

报错提示我们 `if` 和 `else` 返回了不同的类型。如果想要实现返回不同的类型，需要使用下一章节中的**特征对象**。

## 修复上一节中的largest函数

该如何解决编译器报错：

```rust
error[E0369]: binary operation `>` cannot be applied to type `T` // 无法在 `T` 类型上应用`>`运算符
 --> src/main.rs:5:17
  |
5 |         if item > largest {
  |            ---- ^ ------- T
  |            |
  |            T
  |
help: consider restricting type parameter `T` // 考虑使用以下的特征来约束 `T`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> T {
  |             ^^^^^^^^^^^^^^^^^^^^^^
```

在 `largest` 函数体中想要使用大于运算符（`>`）比较两个 `T` 类型的值。这个运算符是标准库中特征 `std::cmp::PartialOrd` 的一个默认方法。所以需要在 `T` 的特征约束中指定 `PartialOrd`。

由于 `PartialOrd` 位于 `prelude` 中所以并不需要通过 `std::cmp` 手动将其引入作用域。所以可以将 `largest` 的签名修改为如下：

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {
```

但是此时编译，又会出现新的错误：

```rust
error[E0508]: cannot move out of type `[T]`, a non-copy slice
 --> src/main.rs:2:23
  |
2 |     let mut largest = list[0];
  |                       ^^^^^^^
  |                       |
  |                       cannot move out of here
  |                       help: consider using a reference instead: `&list[0]`
```

原因是`T`没有实现`Copy`特征。

因此我们只能把所有权进行转移，毕竟只有 `i32` 等基础类型才实现了 `Copy` 特性，可以存储在栈上，而 `T` 可以指代任何实现了`PartialOrd`的类型。

因此，为了让 `T` 拥有 `Copy` 特性，我们可以增加特征约束：

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

如果并不希望限制 `largest` 函数只能用于实现了 `Copy` 特征的类型，可以在`T`特征约束中制定`Clone`特征而不是`Copy`特征。并克隆 `list` 中的每一个值使得 `largest` 函数拥有其所有权。使用 `clone` 函数意味着对于类似 `String` 这样拥有堆上数据的类型，会潜在地分配更多堆上空间，而堆分配在涉及大量数据时可能会相当缓慢。

另一种 `largest` 的实现方式是返回在 `list` 中 `T` 值的引用。如果我们将函数返回值从 `T` 改为 `&T` 并改变函数体使其能够返回一个引用，我们将不需要任何 `Clone` 或 `Copy` 的特征约束而且也不会有任何的堆分配。

## 通过derive派生特征

在之前的代码中，形如 `#[derive(Debug)]` 的代码已经出现了很多次，这种是一种特征派生语法，被 `derive` 标记的对象会自动实现对应的默认特征代码，继承相应的功能。

例如 `Debug` 特征，它有一套自动实现的默认代码，当你给一个结构体标记后，就可以使用 `println!("{:?}", s)` 的形式打印该结构体的对象。

再如 `Copy` 特征，它也有一套自动实现的默认代码，当标记到一个类型上时，可以让这个类型自动实现 `Copy` 特征，进而可以调用 `copy` 方法，进行自我复制。

总之，`derive` 派生出来的是 Rust 默认给我们提供的特征，在开发过程中极大的简化了自己手动实现相应特征的需求，当然，如果有特殊的需求，还可以手动重载该实现。

## 调用方法需要引入特征

在一些场景中，使用 `as` 关键字做类型转换会有比较大的限制，因为你想要在类型转换上拥有完全的控制，例如处理转换错误，那么你将需要 `TryInto`：

```rust
use std::convert::TryInto;

fn main() {
  let a: i32 = 10;
  let b: u16 = 100;

  let b_ = b.try_into()
            .unwrap();

  if a < b_ {
    println!("Ten is less than one hundred.");
  }
}
```

上面代码中引入了 `std::convert::TryInto` 特征，但是却没有使用它，看上去有些困惑，主要原因在于**如果要使用一个特征的方法，那么你需要引入该特征到当前的作用域中**，我们在上面用到了 `try_into` 方法，因此需要引入对应的特征。

但是 Rust 又提供了一个非常便利的办法，即把最常用的标准库中的特征通过 `std::prelude` 模块提前引入到当前作用域中，其中包括了 `std::convert::TryInto`，因此删除第一行的代码 `use ...`，也不会报错。

## 几个综合例子

### 为自定义类型实现 + 操作

在 Rust 中除了数值类型的加法，`String` 也可以做**加法**，因为 Rust 为该类型实现了 `std::ops::Add` 特征，同理，如果我们为自定义类型实现了该特征，那就可以自己实现 `Point1 + Point2` 的操作:

```rust
use std::ops::Add;

// 为Point结构体派生Debug特征，用于格式化输出
#[derive(Debug)]
struct Point<T: Add<T, Output = T>> { //限制类型T必须实现了Add特征，否则无法进行+操作。
    x: T,
    y: T,
}

impl<T: Add<T, Output = T>> Add for Point<T> {
    type Output = Point<T>;

    fn add(self, p: Point<T>) -> Point<T> {
        Point{
            x: self.x + p.x,
            y: self.y + p.y,
        }
    }
}

fn add<T: Add<T, Output=T>>(a:T, b:T) -> T {
    a + b
}

fn main() {
    let p1 = Point{x: 1.1f32, y: 1.1f32};
    let p2 = Point{x: 2.1f32, y: 2.1f32};
    println!("{:?}", add(p1, p2));

    let p3 = Point{x: 1i32, y: 1i32};
    let p4 = Point{x: 2i32, y: 2i32};
    println!("{:?}", add(p3, p4));
}
```

### 自定义类型的打印输出

在开发过程中，往往只要使用 `#[derive(Debug)]` 对我们的自定义类型进行标注，即可实现打印输出的功能：

```rust
#[derive(Debug)]
struct Point{
    x: i32,
    y: i32
}
fn main() {
    let p = Point{x:3,y:3};
    println!("{:?}",p);
}
```

但是在实际项目中，往往需要对自定义类型进行自定义的格式化输出，以让用户更好的阅读理解我们的类型，此时就要为自定义类型实现 `std::fmt::Display` 特征：

```rust
use std::fmt;
use std::fmt::{Display};

#[derive(Debug,PartialEq)]
enum FileState {
  Open,
  Closed,
}

#[derive(Debug)]
struct File {
  name: String,
  data: Vec<u8>,
  state: FileState,
}

impl Display for FileState {
   fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
     match *self {
         FileState::Open => write!(f, "OPEN"),
         FileState::Closed => write!(f, "CLOSED"),
     }
   }
}

impl Display for File {
   fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
      write!(f, "<{} ({})>",
             self.name, self.state)
   }
}

impl File {
  fn new(name: &str) -> File {
    File {
        name: String::from(name),
        data: Vec::new(),
        state: FileState::Closed,
    }
  }
}

fn main() {
  let f6 = File::new("f6.txt");
  //...
  println!("{:?}", f6);
  println!("{}", f6);
}
```



# 特征对象 Trait Object

上节中有段代码无法编译：

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        Post {
           // ...
        }
    } else {
        Weibo {
            // ...
        }
    }
}
```

其中 `Post` 和 `Weibo` 都实现了 `Summary` 特征，因此上面的函数试图通过返回 `impl Summary` 来返回这两个类型，但是编译器却无情地报错了，原因是 `impl Trait` 的返回值类型并不支持多种不同的类型返回。



为了解决上面问题，Rust通过引入：**特征对象(trait object)**

**特征对象**指向实现了特征的类型的实例，这种映射关系是存储在一张表中，可以在运行时通过特征对象找到具体调用的类型方法。

可以通过 `&` 引用或者 `Box<T>` 智能指针的方式来创建特征对象:

```rust
trait Draw {
    fn draw(&self) -> String;
}

impl Draw for u8 {
    fn draw(&self) -> String {
        format!("u8: {}", *self)
    }
}

impl Draw for f64 {
    fn draw(&self) -> String {
        format!("f64: {}", *self)
    }
}

fn draw1(x: Box<dyn Draw>) {
    x.draw();
}

fn draw2(x: &dyn Draw) {
    x.draw();
}

fn main() {
    let x = 1.1f64;
    // do_something(&x);
    let y = 8u8;

    draw1(Box::new(x));
    draw1(Box::new(y));
    draw2(&x);
    draw2(&y);
}

```

上面代码，有几个非常重要的点：

- `draw1` 函数的参数是 `Box<dyn Draw>` 形式的特征对象，该特征对象是通过 `Box::new(x)` 的方式创建的
- `draw2` 函数的参数是 `&dyn Draw` 形式的特征对象，该特征对象是通过 `&x` 的方式创建的
- `dyn` 关键字只用在特征对象的类型声明上，在创建时无需使用 `dyn`



## 特征对象的动态分发

泛型是在编译期完成处理的：编译器会为每一个泛型参数对应的具体类型生成一份代码，这种方式是**静态分发(static dispatch)**，因为是在编译期完成的，对于运行期性能完全没有任何影响。

与静态分发相对应的是**动态分发(dynamic dispatch)**，在这种情况下，直到运行时，才能确定需要调用什么对象的什么方法。之前代码中的关键字 `dyn` 正是在强调这一“动态”的特点。

当使用特征对象时，Rust 必须使用动态分发。下面这张图很好的解释了静态分发 `Box<T>` 和动态分发 `Box<dyn Trait>` 的区别：

![](https://pic1.zhimg.com/80/v2-b771fe4cfc6ebd63d9aff42840eb8e67_1440w.jpg)

结合上文的内容和这张图可以了解：

- **特征对象大小不固定**：这是因为特征对象可能是具体的不同类型，因此特征没有固定大小
- **几乎总是使用特征对象的引用方式**，如 `&dyn Draw`、`Box<dyn Draw>`
  - 虽然特征对象没有固定大小，但它的引用类型的大小是固定的，它由两个指针组成（`ptr` 和 `vptr`），因此占用两个指针大小
  - 一个指针 `ptr` 指向实现了特征 `Draw` 的具体类型的实例，也就是当作特征 `Draw` 来用的类型的实例，比如类型 `Button` 的实例、类型 `SelectBox` 的实例
  - 另一个指针 `vptr` 指向一个虚表 `vtable`，`vtable` 中保存了类型 `Button` 或类型 `SelectBox` 的实例对于可以调用的实现于特征 `Draw` 的方法。当调用方法时，直接从 `vtable` 中找到方法并调用。之所以要使用一个 `vtable` 来保存各实例的方法，是因为实现了特征 `Draw` 的类型有多种，这些类型拥有的方法各不相同，当将这些类型的实例都当作特征 `Draw` 来使用时(此时，它们全都看作是特征 `Draw` 类型的实例)，有必要区分这些实例各自有哪些方法可调用

一定要注意，此时的 `btn` 是 `Draw` 的特征对象的实例，而不再是具体类型 `Button` 的实例，而且 `btn` 的 `vtable` 只包含了实现自特征 `Draw` 的那些方法（比如 `draw`），因此 `btn` 只能调用实现于特征 `Draw` 的 `draw` 方法，而不能调用类型 `Button` 本身实现的方法和类型 `Button` 实现于其他特征的方法。**也就是说，`btn` 是哪个特征对象的实例，它的 `vtable` 中就包含了该特征的方法。**



## Self 与 self

在 Rust 中，有两个`self`，一个指代当前的实例对象，一个指代特征或者方法类型的别名：

```rust
trait Draw {
    fn draw(&self) -> Self;
}

#[derive(Clone)]
struct Button;
impl Draw for Button {
    fn draw(&self) -> Self {
        return self.clone()
    }
}

fn main() {
    let button = Button;
    let newb = button.draw();
}

```

当理解了 `self` 与 `Self` 的区别后，再来看看何为对象安全。



## 特征对象的限制

不是所有特征都能拥有特征对象，只有对象安全的特征才行。当一个特征的所有方法都有如下属性时，它的对象才是安全的：

- 方法的返回类型不能是 `Self`
- 方法没有任何泛型参数

对象安全对于特征对象是必须的，因为一旦有了特征对象，就不再需要知道实现该特征的具体类型是什么了。如果特征方法返回了具体的 `Self` 类型，但是特征对象忘记了其真正的类型，那这个 `Self` 就非常尴尬，因为没人知道它是谁了。

但是对于泛型类型参数来说，当使用特征时其会放入具体的类型参数：此具体类型变成了实现该特征的类型的一部分。而当使用特征对象时其具体类型被抹去了，故而无从得知放入泛型参数类型到底是什么。

标准库中的 `Clone` 特征就不符合对象安全的要求：

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```



如果违反了对象安全的规则，编译器会提示你。例如，如果尝试使用之前的 `Screen` 结构体来存放实现了 `Clone` 特征的类型：

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Clone>>,
}
```

将会得到如下错误：

```rust
error[E0038]: the trait `std::clone::Clone` cannot be made into an object
 --> src/lib.rs:2:5
  |
2 |     pub components: Vec<Box<dyn Clone>>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `std::clone::Clone`
  cannot be made into an object
  |
  = note: the trait cannot require that `Self : Sized`
```

这意味着不能以这种方式使用此特征作为特征对象。



# 进一步深入特征






