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















