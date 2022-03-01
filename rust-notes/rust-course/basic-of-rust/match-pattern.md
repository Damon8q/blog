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
























