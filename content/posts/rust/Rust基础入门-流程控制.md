---
title: "[Rust course]-05 基础入门-流程控制"
date: 2022-05-06T17:10:00+08:00
lastmod: 2022-05-06T17:10:00+08:00
author: nange
draft: false
description: "Rust流程控制"

categories: ["rust"]
series: ["rust-course"]
series_weight: 4
tags: ["rust-notes"]
---

## 流程控制

Rust程序是从上而下顺序执行的，在此过程中，我们可以通过循环、分支等流程控制方式，更好的实现相应的功能。

### 使用`if`做分支控制

```rust
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };

    println!("The value of number is: {}", number);
}
```

* **`if`语句块是表达式**。 这里使用`if`表达式的值给`number`赋值，这里赋值为`5`
* 用`if`来赋值时，要保证每个分支返回的类型一样。否则编译报错。

### 使用`else if`来处理多重条件

```rust
fn main() {
    let n = 6;

    if n % 4 == 0 {
        println!("number is divisible by 4");
    } else if n % 3 == 0 {
        println!("number is divisible by 3");
    } else if n % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

有一点要注意，就算有多个分支能匹配，也只有第一个匹配的分支会被执行！

如果代码中有大量的 `else if`会让代码变得极其丑陋，下一章的 `match` 专门用以解决多分支模式匹配的问题。

### 循环控制

在 Rust 语言中有三种循环方式：`for`、`while` 和 `loop`，其中 `for` 循环是 Rust 循环王冠上的明珠。

* **for循环** 大杀器
  
  ```rust
  fn main() {
      for i in 1..=5 {
          println!("{}", i);
      }
  }
  ```
  
  以上代码核心就在于 `for` 和 `in` 的联动，语义表达如下：
  
  ```rust
  for 元素 in 集合 {
    // do something...
  }
  ```
  
  注意，使用 `for` 时我们往往使用集合的引用形式，除非你不想在后面的代码中继续使用该集合（比如我们这里使用了 `container` 的引用）。如果不使用引用的话，所有权会被转移（move）到 `for` 语句块中，后面就无法再使用这个集合了)：
  
  ```rust
  for item in &container {
    // ...
  }
  ```
  
  > 对于实现了 `copy` 特征的数组(例如 [i32; 10] )而言， `for item in arr` 并不会把 `arr` 的所有权转移，而是直接对其进行了拷贝，因此循环之后仍然可以使用 `arr` 。
  
  如果想在循环中，**修改该元素**，可以使用 `mut` 关键字：
  
  ```rust
  for item in &mut collection {
    // ...
  }
  ```
  
  总结如下：

| 使用方法                          | 等价使用方式                                            | 所有权   |
| ----------------------------- | ------------------------------------------------- | ----- |
| `for item in collection`      | `for item in IntoIterator::into_iter(collection)` | 转移所有权 |
| `for item in &collection`     | `for item in collection.iter()`                   | 不可变借用 |
| `for item in &mut collection` | `for item in collection.iter_mut()`               | 可变借用  |

  如果想在循环中**获取元素的索引**：

```rust
fn main() {
    let a = [4,3,2,1];
    // `.iter()` 方法把 `a` 数组变成一个迭代器
    for (i,v) in a.iter().enumerate() {
        println!("第{}个元素是{}",i+1,v);
    }
}
```

  如果想用 `for` 循环控制某个过程执行 10 次，但是又不想单独声明一个变量来控制这个流程:

```rust
for _ in 0..10 {
  // ...
}
```

  可以用 `_` 来替代 `i` 用于 `for` 循环中，在 Rust 中 `_` 的含义是忽略该值或者类型的意思，如果不使用 `_`，那么编译器会给一个 `变量未使用的` 的警告。

  **两种循环方式优劣对比**:

```rust
// 第一种
let collection = [1, 2, 3, 4, 5];
for i in 0..collection.len() {
  let item = collection[i];
  // ...
}

// 第二种
for item in collection {
  // ... 
}
```

  第一种方式是循环索引，然后通过索引下标去访问集合，第二种方式是直接循环集合中的元素，优劣如下：

* **性能**：第一种使用方式中 `collection[index]` 的索引访问，会因为边界检查(Bounds Checking)导致运行时的性能损耗 —— Rust会检查并确认 `index` 是否落在集合内，但是第二种直接迭代的方式就不会触发这种检查，因为编译器会在编译时就完成分析并证明这种访问是合法的

* **安全**： 第一种方式里对 `collection` 的索引访问是非连续的，存在一定可能性在两次访问之间，`collection` 发生了变化，导致脏数据产生。而第二种直接迭代的方式是连续访问，因此不存在这种风险（这里是因为所有权吗？是的话可能要强调一下）
  
  由于 `for` 循环无需任何条件限制，也不需要通过索引来访问，因此是最安全也是最常用的，通过与下面的 `while` 的对比，我们能看到为什么 `for` 会更加安全。
  
  **continue**
  使用 `continue` 可以跳过当前当次的循环，开始下次的循环：

```rust
for i in 1..4 {
     if i == 2 {
         continue;
     }
     println!("{}",i);
 }
```

  **break**
  使用 `break` 可以直接跳出当前整个循环：

```rust
 for i in 1..4 {
     if i == 2 {
         break;
     }
     println!("{}",i);
 }
```

* **`while`循环**
  如果你需要一个条件来循环，当该条件为 `true` 时，继续循环，条件为 `false`，跳出循环，那么 `while` 就非常适用：
  
  ```rust
  fn main() {
      let mut n = 0;
      while n <= 5  {
          println!("{}!", n);
          n = n + 1;
      }
  
      println!("我出来了！");
  }
  ```
  
  也可以使用`loop`实现相同的功能，但是`while`要简洁很多。
  
  **`while` vs `for`**
  使用`while`实现`for`的功能：
  
  ```rust
  fn main() {
      let a = [10, 20, 30, 40, 50];
      let mut index = 0;
  
      while index < 5 {
          println!("the value is: {}", a[index]);
          index = index + 1;
      }
  }
  ```
  
  数组中的所有五个元素都如期被打印出来。但这个过程很容易出错；如果索引长度不正确会导致程序 ***panic***。这也使程序更慢，因为编译器增加了运行时代码来对每次循环的每个元素进行条件检查。
  
  `for`循环代码如下：
  
  ```rust
  fn main() {
      let a = [10, 20, 30, 40, 50];
  
      for element in a.iter() {
          println!("the value is: {}", element);
      }
  }
  ```
  
  可以看出，`for` 并不会使用索引去访问数组，因此更安全也更简洁，同时避免 `运行时的边界检查`，性能更高。

* **`loop`循环**
  
  对于循环而言，`loop` 循环毋庸置疑，是适用面最高的，它可以适用于所有循环场景（虽然能用，但是在很多场景下， `for` 和 `while` 才是最优选择），因为 `loop` 就是一个简单的无限循环，你可以在内部实现逻辑通过 `break` 关键字来控制循环何时结束。示例：
  
  ```rust
  fn main() {
      let mut counter = 0;
      let result = loop {
          counter += 1;
          if counter == 10 {
              break counter * 2;
          }
      };
  
      println!("The result is {}", result);
  }
  ```
  
  以上代码当 `counter` 递增到 `10` 时，就会通过 `break` 返回一个 `counter * 2` 的值，最后赋给 `result` 并打印出来。
  
  几个注意点：
  
  * **`break`可以单独使用，也可以带一个返回值**， 有些类似`return`
  * **`loop`是一个表达式** ，因此可以返回一个值
