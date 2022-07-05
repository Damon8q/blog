---
title: "[Rust course]-09 基础入门-集合类型"
date: 2022-05-06T17:49:00+08:00
lastmod: 2022-05-06T17:49:00+08:00
author: nange
draft: false
description: "Rust集合类型"

categories: ["编程语言"]
series: ["rust-course"]
series_weight: 9
tags: ["rust"]
---

## 动态数组 Vector

动态数组允许存储多个值，这些值在内存中一个紧挨着另一个排列，因此访问其中某个元素的成本非常低。动态数组只能存储相同类型的元素，如果想存储不同类型的元素，可以使用枚举类型或者特征对象。

## 创建动态数组

* **Vec::new**
  
  ```rust
  let v: Vec<i32> = Vec::new();
  ```
  
  向`v`中增加一个元素，则无需手动声明类型：
  
  ```rust
  let mut v = Vec::new();
  v.push(1);
  ```
  
  因为编译器通过 `v.push(1)`，推测出 `v` 中的元素类型是 `i32`，因此推导出 `v` 的类型是 `Vec<i32>`。
  
  > 如果预先知道要存储的元素个数，可以使用 `Vec::with_capacity(capacity)` 创建动态数组，这样可以避免因为插入大量新数据导致频繁的内存分配和拷贝，提升性能

* **vec![]**
  
  使用宏 `vec!` 来创建数组，与 `Vec::new` 有所不同，前者能在创建同时给予初始化值：
  
  ```rust
  let v = vec![1, 2, 3];
  ```
  
  此处也无需指定类型，编译器能推断出是`Vec<i32>`类型。

### 更新Vector

向尾部添加元素：

```rust
let mut v = Vec::new();
v.push(1);
```

与其它类型一样，必须将 `v` 声明为 `mut` 后，才能进行修改。

### Vector与其元素共存亡

跟结构体一样，`Vector` 类型在超出作用域范围后，会被自动删除：

```rust
{
    let v = vec![1, 2, 3];

    // ...
} // <- v超出作用域并在此处被删除
```

当 `Vector` 被删除后，它内部存储的所有内容也会随之被删除。

### 从Vector中读取元素

读取指定位置元素的两种方式：

```rust
let v = vec![1, 2, 3, 4, 5];

let third: &i32 = &v[2];
println!("第三个元素是 {}", third);

match v.get(2) {
    Some(third) => println!("第三个元素是 {}", third),
    None => println!("去你的第三个元素，根本没有！"),
}
```

和其它语言一样，集合类型的索引下标都是从 `0` 开始，`&v[2]` 表示借用 `v` 中的第三个元素，最终会获得该元素的引用。而 `v.get(2)` 也是访问第三个元素，但是有所不同的是，它返回了 `Option<&T>`，因此还需要额外的 `match` 来匹配解构出具体的值。

**下标索引与get的区别：**

```rust
let v = vec![1, 2, 3, 4, 5];

let does_not_exist = &v[100];
let does_not_exist = v.get(100);
```

运行以上代码，`&v[100]` 的访问方式会导致程序无情报错退出，因为发生了数组越界访问。 但是 `v.get` 就不会，它在内部做了处理，有值的时候返回 `Some(T)`，无值的时候返回 `None`，因此 `v.get` 的使用方式非常安全。

为何不统一使用 `v.get` 的形式？因为实在是有些啰嗦，Rust 语言的设计者和使用者在审美这方面还是相当统一的：简洁即正义，何况性能上也会有轻微的损耗。

选择：当确定索引不会越界的时候，就用索引访问，否则用 `.get`。

**同时借用多个数组元素：**

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];

v.push(6);

println!("The first element is: {}", first);
```

来推断下结果，首先 `first = &v[0]` 进行了不可变借用，`v.push` 进行了可变借用，如果 `first` 在 `v.push` 之后不再使用，那么该段代码可以成功编译。

可是上面的代码中，`first` 这个不可变借用在可变借用 `v.push` 后被使用了，那么编译器就会报错：

```rust
Compiling collections v0.1.0 (file:///projects/collections)
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable 无法对v进行可变借用，因此之前已经进行了不可变借用
--> src/main.rs:6:5
|
4 |     let first = &v[0];
|                  - immutable borrow occurs here // 不可变借用发生在此处
5 |
6 |     v.push(6);
|     ^^^^^^^^^ mutable borrow occurs here // 可变借用发生在此处
7 |
8 |     println!("The first element is: {}", first);
|                                          ----- immutable borrow later used here // 不可变借用在这里被使用

For more information about this error, try `rustc --explain E0502`.
error: could not compile `collections` due to previous error
```

按理来说，这两个引用不应该互相影响的：一个是查询元素，一个是在数组尾部插入元素，完全不相干的操作，为何编译器要这么严格呢？

原因在于：数组的大小是可变的，当旧数组的大小不够用时，Rust 会重新分配一块更大的内存空间，然后把旧数组拷贝过来。这种情况下，之前的引用显然会指向一块无效的内存，这非常 rusty —— 对用户进行严格的教育。

### 迭代遍历Vector中的元素

如果想要依次访问数组中的元素，可以使用迭代的方式去遍历数组，这种方式比用下标的方式去遍历数组更安全也更高效（每次下标访问都会触发数组边界检查）：

```rust
let v = vec![1, 2, 3];
for i in &v {
    println!("{}", i);
}
```

也可以在迭代过程中，修改 `Vector` 中的元素：

```rust
let mut v = vec![1, 2, 3];
for i in &mut v {
    *i += 10
}
```

### 存储不同类型的元素

* 通过枚举
  
  ```rust
  #[derive(Debug)]
  enum IpAddr {
      V4(String),
      V6(String)
  }
  fn main() {
      let v = vec![
          IpAddr::V4("127.0.0.1".to_string()),
          IpAddr::V6("::1".to_string())
      ];
  
      for ip in v {
          show_addr(ip)
      }
  }
  
  fn show_addr(ip: IpAddr) {
      println!("{:?}",ip);
  }
  ```
  
  数组 `v` 中存储了两种不同的 `ip` 地址，但是这两种都属于 `IpAddr` 枚举类型的成员，因此可以存储在数组中。

* 通过特征对象
  
  ```rust
  trait IpAddr {
      fn display(&self);
  }
  
  struct V4(String);
  impl IpAddr for V4 {
      fn display(&self) {
          println!("ipv4: {:?}",self.0)
      }
  }
  struct V6(String);
  impl IpAddr for V6 {
      fn display(&self) {
          println!("ipv6: {:?}",self.0)
      }
  }
  
  fn main() {
      let v: Vec<Box<dyn IpAddr>> = vec![
          Box::new(V4("127.0.0.1".to_string())),
          Box::new(V6("::1".to_string())),
      ];
  
      for ip in v {
          ip.display();
      }
  }
  ```
  
  为 `V4` 和 `V6` 都实现了特征 `IpAddr`，然后将它俩的实例用 `Box::new` 包裹后，存在了数组 `v` 中，需要注意的是，这里必需手动的指定类型：`Vec<Box<dyn IpAddr>>`，表示数组 `v` 存储的是特征 `IpAddr` 的对象，这样就实现了在数组中存储不同的类型。
  
  在实际使用场景中，特征对象数组要比枚举数组常见很多，主要原因在于**特征对象**非常灵活，而编译器对枚举的限制较多，且无法动态增加类型。

## KV存储 HashMap

和动态数组一样，`HashMap` 也是 Rust 标准库中提供的集合类型，但是又与动态数组不同，`HashMap` 中存储的是一一映射的 `KV` 键值对，并提供了平均复杂度为 `O(1)` 的查询方法。

### 创建HashMap

* **使用`new`方法创建**
  
  ```rust
  use std::collections::HashMap;
  
  // 创建一个HashMap，用于存储宝石种类和对应的数量
  let mut my_gems = HashMap::new();
  
  // 将宝石类型和对应的数量写入表中
  my_gems.insert("红宝石", 1);
  my_gems.insert("蓝宝石", 2);
  my_gems.insert("河边捡的误以为是宝石的破石头", 18);
  ```
  
  其HashMap的类型，Rust能自动推断出是：`HashMap<&str,i32>`。
  
  所有的集合类型都是动态的，它们没有固定的内存大小，因此它们底层的数据都存储在内存堆上，然后通过一个存储在栈中的引用类型来访问。同时，跟其它集合类型一致，`HashMap` 也是内聚性的，即所有的 `K` 必须拥有同样的类型，`V` 也是如此。
  
  > 跟 `Vec` 一样，如果预先知道要存储的 `KV` 对个数，可以使用 `HashMap::with_capacity(capacity)` 创建指定大小的 `HashMap`，避免频繁的内存分配和拷贝，提升性能。

* **使用迭代器和collect方法创建**
  
  在实际使用中，不是所有的场景都能 `new` 一个哈希表后，然后依次插入对应的键值对，而是可能会从另外一个数据结构中，获取到对应的数据，最终生成 `HashMap`。
  
  例如把`Vec<(String, u32)>`原始类型转化为`HashMap<String, u32>`，我们虽然可以这样：
  
  ```rust
  fn main() {
      use std::collections::HashMap;
  
      let teams_list = vec![
          ("中国队".to_string(), 100),
          ("美国队".to_string(), 10),
          ("日本队".to_string(), 50),
      ];
  
      let mut teams_map = HashMap::new();
      for team in &teams_list {
          teams_map.insert(&team.0, team.1);
      }
  
      println!("{:?}",teams_map)
  }
  ```
  
  这样可行，但是不够rusty。
  
  Rust 为我们提供了一个非常精妙的解决办法：先将 `Vec` 转为迭代器，接着通过 `collect` 方法，将迭代器中的元素收集后，转成 `HashMap`：
  
  ```rust
  fn main() {
      use std::collections::HashMap;
  
      let teams_list = vec![
          ("中国队".to_string(), 100),
          ("美国队".to_string(), 10),
          ("日本队".to_string(), 50),
      ];
  
      let teams_map: HashMap<_,_> = teams_list.into_iter().collect();
  
      println!("{:?}",teams_map)
  }
  ```
  
  `into_iter` 方法将列表转为迭代器，接着通过 `collect` 进行收集，不过需要注意的是，`collect` 方法在内部实际上支持生成多种类型的目标集合，因此需要通过类型标注 `HashMap<_,_>` 来告诉编译器：请帮我们收集为 `HashMap` 集合类型，具体的 `KV` 类型，麻烦编译器帮我们推导。

### 所有权转移

`HashMap` 的所有权规则与其它 Rust 类型没有区别：

* 若类型实现 `Copy` 特征，该类型会被复制进 `HashMap`，因此无所谓所有权
* 若没实现 `Copy` 特征，所有权将被转移给 `HashMap` 中

例如：

```rust
fn main() {
    use std::collections::HashMap;

    let name = String::from("Sunface");
    let age = 18;

    let mut handsome_boys = HashMap::new();
    handsome_boys.insert(name, age);

    println!("因为过于无耻，{}已经被从帅气男孩名单中除名", name);
    println!("还有，他的真实年龄远远不止{}岁", age);
}
```

运行代码，报错如下：

```rust
error[E0382]: borrow of moved value: `name`
  --> src/main.rs:10:32
```

`name` 是 `String` 类型，因此它受到所有权的限制，在 `insert` 时，它的所有权被转移给 `handsome_boys`，所以最后在使用时，会遇到这个报错。

**如果你使用引用类型放入 HashMap 中**，请确保该引用的生命周期至少跟 `HashMap` 活得一样久。或者可以把`name` 克隆一次，作为key：`
 handsome_boys.insert(name.clone(), age)`。

### 查询 HashMap

通过`get`方法获取元素：

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let team_name = String::from("Blue");
let score: Option<&i32> = scores.get(&team_name);
```

上面有几点需要注意：

* `K` 是 `String`类型，但是 `get`方法参数是`&str`类型（也就是其引用）

* `get` 方法返回一个 `Option<&i32>` 类型：当查询不到时，会返回一个 `None`，查询到时返回 `Some(&i32)`

* `&i32` 是对 `HashMap` 中值的借用，如果不使用借用，可能会发生所有权的转移

还可以通过循环的方式依次遍历 `KV` 对：

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

for (key, value) in &scores {
    println!("{}: {}", key, value);
}
```

### 更新HashMap中的值

```rust
fn main() {
    use std::collections::HashMap;

    let mut scores = HashMap::new();

    scores.insert("Blue", 10);

    // 覆盖已有的值
    let old = scores.insert("Blue", 20);
    assert_eq!(old, Some(10));

    // 查询新插入的值
    let new = scores.get("Blue");
    assert_eq!(new, Some(&20));

    // 查询Yellow对应的值，若不存在则插入新值
    let v = scores.entry("Yellow").or_insert(5);
    assert_eq!(*v, 5); // 不存在，插入5

    // 查询Yellow对应的值，若不存在则插入新值
    let v = scores.entry("Yellow").or_insert(50);
    assert_eq!(*v, 5); // 已经存在，因此50没有插入
}
```

**在已有值的基础上更新**

另一个常用场景如下：查询某个 `key` 对应的值，若不存在则插入新值，若存在则对已有的值进行更新，例如在文本中统计词语出现的次数：

```rust
use std::collections::HashMap;

let text = "hello world wonderful world";

let mut map = HashMap::new();
// 根据空格来切分字符串(英文单词都是通过空格切分)
for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1;
}

println!("{:?}", map);
```

有两点值得注意：

* `or_insert` 返回了 `&mut v` 引用，因此可以通过该可变引用直接修改 `map` 中对应的值
* 使用 `count` 引用时，需要先进行解引用 `*count`，否则会出现类型不匹配

### 哈希函数

如果要实现 `Key` 与 `Value` 的一一对应，意味着我们要能比较两个 `Key` 的相等性。因此，一个类型能否作为 `Key` 的关键就是是否能进行相等比较，或者说该类型是否实现了 `std::cmp::Eq` 特征。

> f32 和 f64 浮点数，没有实现 `std::cmp::Eq` 特征，因此不可以用作 `HashMap` 的 `Key`

若一个复杂点的类型作为 `Key`，那怎么在底层对它进行存储，怎么使用它进行查询和比较？ 好在有哈希函数：通过它把 `Key` 计算后映射为哈希值，然后使用该哈希值来进行存储、查询、比较等操作。

如何保证不同 `Key` 通过哈希后的两个值不会相同？如果相同，那意味着我们使用不同的 `Key`，却查到了同一个结果， 此时，就涉及到安全性跟性能的取舍了。

若要追求安全，尽可能减少冲突，同时防止拒绝服务（Denial of Service, DoS）攻击，就要使用密码学安全的哈希函数，`HashMap` 就是使用了这样的哈希函数。反之若要追求性能，就需要使用没有那么安全的算法。

**高性能第三方库**

若性能测试显示当前标准库默认的哈希函数不能满足你的性能需求，就需要去 [`crates.io`](https://crates.io/) 上寻找其它的哈希函数实现，使用方法很简单：

```rust
use std::hash::BuildHasherDefault;
use std::collections::HashMap;
// 引入第三方的哈希函数
use twox_hash::XxHash64;

// 指定HashMap使用第三方的哈希函数XxHash64
let mut hash: HashMap<_, _, BuildHasherDefault<XxHash64>> = HashMap::default();
hash.insert(42, "the answer");
assert_eq!(hash.get(&42), Some(&"the answer"));
```

> 目前，`HashMap` 使用的哈希函数是 `SipHash`，它的性能不是很高，但是安全性很高。`SipHash` 在中等大小的 `Key` 上，性能相当不错，但是对于小型的 `Key` （例如整数）或者大型 `Key` （例如字符串）来说，性能还是不够好。若你需要极致性能，例如实现算法，可以考虑这个库：[ahash](https://github.com/tkaitchuck/ahash)
