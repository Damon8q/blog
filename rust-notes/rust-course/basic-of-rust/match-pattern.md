# 模式匹配

模式匹配经常出现在函数式编程里，用于为复杂的类型系统提供轻松地结构能力。

在枚举和流程控制章节，遗留了两个问题，一是如何对`Option`枚举进一步处理，另一个式如何用`match`来代替`else if`这种丑陋的多重分支使用方式。接下来就会解决此问题。

# match和if let

在Rust中，最常用的就是`match`和`if let`，先看一个`match`例子：

```rust
enum Direction {
    East,
    West,
    North,
    South,
}

fn main() {
    let dire = Direction::South;
    match dire {
        Direction::East => println!("East"),
        Direction::North | Direction::South => {
            println!("South or North");
        },
        _ => println!("West"),
    };
}
```

几点注意：

* `match`匹配必须要穷举所有可能，可以用`_`来代表未列出的所有可能性

* `match`的每个分支都必须式一个表达式，且所有分支表达式的返回值类型必须相同

* **`X|Y`**，是逻辑运算符`或`，代表该分支可以匹配`X`也可以匹配`Y`，只要满足一个即可

其实 `match` 跟其他语言中的 `switch` 有些像，`_` 类似于 `switch` 中的 `default`。



## match匹配

`match`的通用形式：

```rust
match target {
    模式1 => 表达式1,
    模式2 => {
        语句1;
        语句2;
        表达式2
    },
    _ => 表达式3
}
```

一个分支有两个部分：**一个模式和针对该模式的处理代码**。每个分支相关联的代码是一个表达式，而表达式的结果值将作为整个 `match` 表达式的返回值。如果分支有多行代码，那么需要用 `{}` 包裹，同时最后一行代码需要是一个表达式。

* **使用`match`表达式赋值**
  `match`本身也是一个表达式，因此可以用它来赋值：

  ```rust
  enum IpAddr {
     Ipv4,
     Ipv6
  }
  
  fn main() {
      let ip1 = IpAddr::Ipv6;
      let ip_str = match ip1 {
          IpAddr::Ipv4 => "127.0.0.1",
          _ => "::1",
      };
  
      println!("{}", ip_str);
  }
  ```

  这里匹配到 `_` 分支，所以将 `"::1"` 赋值给了 `ip_str`。

* **模式绑定**
  模式匹配的另外一个重要功能是从模式中取出绑定的值，例如：

  ```rust
  enum Action {
      Say(String),
      MoveTo(i32, i32),
      ChangeColorRGB(u16, u16, u16),
  }
  
  fn main() {
      let actions = [ 
          Action::Say("Hello Rust".to_string()),
          Action::MoveTo(1,2),
          Action::ChangeColorRGB(255,255,0),
      ];
      for action in actions {
          match action {
              Action::Say(s) => {
                  println!("{}", s);
              },
              Action::MoveTo(x, y) => {
                  println!("point from (0, 0) move to ({}, {})", x, y);
              },
              Action::ChangeColorRGB(r, g, _) => {
                  println!("change color into '(r:{}, g:{}, b:0)', 'b' has been ignored",
                      r, g,
                  );
              }
          }
      }
  }
  ```

  运行：

  ```rust
  Hello Rust
  point from (0, 0) move to (1, 2)
  change color into '(r:255, g:255, b:0)', 'b' has been ignored
  ```

* **穷尽匹配**
  `match`的匹配必须穷尽所有情况，例如：

  ```rust
  enum Direction {
      East,
      West,
      North,
      South,
  }
  
  fn main() {
      let dire = Direction::South;
      match dire {
          Direction::East => println!("East"),
          Direction::North | Direction::South => {
              println!("South or North");
          },
      };
  }
  ```

  没有处理 `Direction::West` 的情况，因此会报错：

  ```rust
  error[E0004]: non-exhaustive patterns: `West` not covered
  ```

* **_通配符**
  当我们不想列出所有匹配值时，可以使用一个特殊模式:

  ```rust
  let some_u8_value = 0u8;
  match some_u8_value {
      1 => println!("one"),
      3 => println!("three"),
      5 => println!("five"),
      7 => println!("seven"),
      _ => (),
  }
  ```

  通过将 `_` 其放置于其他分支后，`_` 将会匹配所有遗漏的值。`()` 表示返回**单元类型**与所有分支返回值的类型相同，所以当匹配到 `_` 后，什么也不会发生。

  然后，在某些场景下，我们其实只关心**某一个值是否存在**，此时 `match` 就显得过于啰嗦。这时候需要使用`if let`匹配。



## if let 匹配

有时会遇到只有一个模式的值需要被处理，其它值直接忽略的场景，如果用 `match` 来处理就要写成下面这样：

```rust
let v = Some(3u8);
match v {
    Some(3) => println!("three"),
    _ => (),
}
```

使用`if let`来实现更加简洁：

```rust
if let Some(3) = v {
    println!("three");
}
```

**当只要匹配一个条件，且忽略(或者可统一处理)其他条件时就用 `if let` ，否则都用 `match`**。



## matches! 宏

Rust 标准库中提供了一个非常实用的宏：`matches!`，它可以将一个表达式跟模式进行匹配，然后返回匹配的结果 `true` or `false`。

例如，有一个动态数组，里面存有以下枚举：

```rust
enum MyEnum {
    Foo,
    Bar
}

fn main() {
    let v = vec![MyEnum::Foo,MyEnum::Bar,MyEnum::Foo];
}
```

现在如果想对 `v` 进行过滤，只保留类型是 `MyEnum::Foo` 的元素，可以这么写：

```rust
v.iter().filter(|x| x == MyEnum::Foo);
```

但是，实际上这行代码会报错，因为你无法将 `x` 直接跟一个枚举成员进行比较。当然可以使用 `match` 来完成，但是会导致代码更为啰嗦，是否有更简洁的方式？答案是使用 `matches!`：

```rust
v.iter().filter(|x| matches!(x, MyEnum::Foo));
```

很简单也很简洁，更多的例子：

```rust
let foo = 'f';
assert!(matches!(foo, 'A'..='Z' | 'a'..='z'));

let bar = Some(4);
assert!(matches!(bar, Some(x) if x > 2));

```



## 变量覆盖

无论是是 `match` 还是 `if let`，他们都可以在模式匹配时覆盖掉老的值，绑定新的值:

```rust
fn main() {
   let age = Some(30);
   println!("在匹配前，age是{:?}",age);
   if let Some(age) = age {
       println!("匹配出来的age是{}",age);
   }

   println!("在匹配后，age是{:?}",age);
}
```

输出：

```rust
在匹配前，age是Some(30)
匹配出来的age是30
在匹配后，age是Some(30)
```

可以看出在 `if let` 中，`=` 右边 `Some(i32)` 类型的 `age` 被左边 `i32` 类型的新 `age` 覆盖了，该覆盖一直持续到 `if let` 语句块的结束。因此第三个 `println!` 输出的 `age` 依然是 `Some(i32)` 类型。

对于 `match` 类型也是如此:

```rust
fn main() {
   let age = Some(30);
   println!("在匹配前，age是{:?}",age);
   match age {
       Some(age) =>  println!("匹配出来的age是{}",age),
       _ => ()
   }
   println!("在匹配后，age是{:?}",age);
}

```

需要注意的是，**`match` 中的变量覆盖其实不是那么的容易看出**，因此要小心！



# 解构Option

`Option`枚举，是用来解决Rust变量是否有值的问题，定义如下：

```rust'
enum Option<T> {
	Some(T),
	None,
}
```

简单说就是：**一个变量要么有值：Some(T)，要么为空：None。**

> 因为 `Option`，`Some`，`None` 都包含在 `prelude` 中，因此可以直接通过名称来使用它们，而无需以 `Option::Some` 这种形式，不要因为调用路径变短了，就忘记 `Some` 和 `None` 也是 `Option` 底下的枚举成员！



## 匹配 Option<T>

使用 `Option<T>`，是为了从 `Some` 中取出其内部的 `T` 值以及处理没有值的情况。例如：

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);	
let six = plus_one(five);	// return Some(6)
let none = plus_one(None);	// return None
```



# 模式适用场景

## 模式

模式是 Rust 中的特殊语法，它用来匹配类型中的结构和数据，它往往和 `match` 表达式联用，以实现强大的模式匹配能力。模式一般由以下内容组合而成：

- 字面值
- 解构的数组、枚举、结构体或者元组
- 变量
- 通配符
- 占位符

## 可能用到模式的地方

* **match分支**

  ```rust
  match VALUE {
      PATTERN => EXPRESSION,
      PATTERN => EXPRESSION,
      PATTERN => EXPRESSION,
  }
  ```

  如上所示，`match` 的每个分支就是一个**模式**。
  

* **if let 分支**
  `if let` 往往用于匹配一个模式，而忽略剩下的所有模式的场景：

  ```rust
  if let PATTERN = SOME_VALUE {
  	// do sth...
  }
  ```



* **while let 条件循环**

  一个与 `if let` 类似的结构是 `while let` 条件循环，它允许只要模式匹配就一直进行 `while` 循环。例如：

  ```rust
  // Vec是动态数组
  let mut stack = Vec::new();
  
  // 向数组尾部插入元素
  stack.push(1);
  stack.push(2);
  stack.push(3);
  
  // stack.pop从数组尾部弹出元素
  while let Some(top) = stack.pop() {
      println!("{}", top);
  }
  ```

  对于 `while` 来说，只要 `pop` 返回 `Some` 就会一直不停的循环。一旦其返回 `None`，`while` 循环停止。我们可以使用 `while let` 来弹出栈中的每一个元素。

  也可以用 `loop` + `if let` 或者 `match` 来实现这个功能，但是会更加啰嗦。

  

* **for 循环**

  ```rust
  let v = vec!['a', 'b', 'c'];
  
  for (index, value) in v.iter().enumerate() {
      println!("{} is at index {}", value, index);
  }
  ```

  这里使用 `enumerate` 方法产生一个迭代器，该迭代器每次迭代会返回一个 `(索引，值)` 形式的元组，然后用 `(index,value)` 来匹配。

  

* **let 语句**

  ```rust
  let PATTERN = EXPRESSION;
  //例如：
  // let x = 5;
  ```

  这其中，`x` 也是一种模式绑定，代表将**匹配的值绑定到变量x上**。因此，在 Rust 中,**变量名也是一种模式**，只不过它比较朴素很不起眼罢了。

  

  ```rust
  let (x, y, z) = (1, 2, 3);
  ```

  上面将一个元组与模式进行匹配(**模式和值的类型必需相同！**)，然后把 `1, 2, 3` 分别绑定到 `x, y, z` 上。
  

  模式匹配要求两边的类型必须相同，否则就会导致下面的报错：

  ```rust
  let (x, y) = (1, 2, 3);
  ```

  ```rust
  error[E0308]: mismatched types
   --> src/main.rs:4:5
    |
  4 | let (x, y) = (1, 2, 3);
    |     ^^^^^^   --------- this expression has type `({integer}, {integer}, {integer})`
    |     |
    |     expected a tuple with 3 elements, found one with 2 elements
    |
    = note: expected tuple `({integer}, {integer}, {integer})`
               found tuple `(_, _)`
  For more information about this error, try `rustc --explain E0308`.
  error: could not compile `playground` due to previous error
  ```

  对于元组来说，元素个数也是类型的一部分！

  

* **函数参数**

  函数参数也是模式：

  ```rust
  fn foo(x: i32) {
      // 代码
  }
  ```

  其中 `x` 就是一个模式，你还可以在参数中匹配元组：

  ```rust
  fn print_coordinates(&(x, y): &(i32, i32)) {
      println!("Current location: ({}, {})", x, y);
  }
  
  fn main() {
      let point = (3, 5);
      print_coordinates(&point);
  }
  ```

  `&(3, 5)` 会匹配模式 `&(x, y)`，因此 `x` 得到了 `3`，`y` 得到了 `5`。

  

* **if 和 if let**

  对于以下代码，编译器会报错：

  ```rust
  let Some(x) = some_option_value;
  ```

  因为右边的值不一定是`Some`，也有可能是`None`，也就是上面代码遗漏了`None`的匹配。

  类似 `let` 和 `for`、`match` 都必须要求完全覆盖匹配，才能通过编译。

  

  但是对于 `if let`，就可以这样使用：

  ```rust
  if let Some(x) = some_option_value {
      println!("{}", x);
  }
  ```

  因为 `if let` 允许匹配一种模式，而忽略其余的模式。



# 全模式列表













