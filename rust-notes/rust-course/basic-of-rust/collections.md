# 动态数组 Vector

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
  
  因为编译器通过 `v.push(1)`，推测出 `v` 中的元素类型是 `i32`，因此推导出 `v` 的类型是 `Vec<i32>`。
  
  > 如果预先知道要存储的元素个数，可以使用 `Vec::with_capacity(capacity)` 创建动态数组，这样可以避免因为插入大量新数据导致频繁的内存分配和拷贝，提升性能

* **vec![]**
  
  使用宏 `vec!` 来创建数组，与 `Vec::new` 有所不同，前者能在创建同时给予初始化值：
  
  ```rust
  let v = vec![1, 2, 3];
  ```
  
  此处也无需指定类型，编译器能推断出是`Vec<i32>`类型。



## 更新Vector

向尾部添加元素：

```rust
let mut v = Vec::new();
v.push(1);
```

与其它类型一样，必须将 `v` 声明为 `mut` 后，才能进行修改。



## Vector与其元素共存亡

跟结构体一样，`Vector` 类型在超出作用域范围后，会被自动删除：

```rust
{
    let v = vec![1, 2, 3];

    // ...
} // <- v超出作用域并在此处被删除
```

当 `Vector` 被删除后，它内部存储的所有内容也会随之被删除。



## 从Vector中读取元素

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

和其它语言一样，集合类型的索引下标都是从 `0` 开始，`&v[2]` 表示借用 `v` 中的第三个元素，最终会获得该元素的引用。而 `v.get(2)` 也是访问第三个元素，但是有所不同的是，它返回了 `Option<&T>`，因此还需要额外的 `match` 来匹配解构出具体的值。



**下标索引与get的区别：**

```rust
let v = vec![1, 2, 3, 4, 5];

let does_not_exist = &v[100];
let does_not_exist = v.get(100);
```

运行以上代码，`&v[100]` 的访问方式会导致程序无情报错退出，因为发生了数组越界访问。 但是 `v.get` 就不会，它在内部做了处理，有值的时候返回 `Some(T)`，无值的时候返回 `None`，因此 `v.get` 的使用方式非常安全。

为何不统一使用 `v.get` 的形式？因为实在是有些啰嗦，Rust 语言的设计者和使用者在审美这方面还是相当统一的：简洁即正义，何况性能上也会有轻微的损耗。

选择：当确定索引不会越界的时候，就用索引访问，否则用 `.get`。



**同时借用多个数组元素：**

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];

v.push(6);

println!("The first element is: {}", first);
```

来推断下结果，首先 `first = &v[0]` 进行了不可变借用，`v.push` 进行了可变借用，如果 `first` 在 `v.push` 之后不再使用，那么该段代码可以成功编译。

可是上面的代码中，`first` 这个不可变借用在可变借用 `v.push` 后被使用了，那么编译器就会报错：

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



## 迭代遍历Vector中的元素

如果想要依次访问数组中的元素，可以使用迭代的方式去遍历数组，这种方式比用下标的方式去遍历数组更安全也更高效（每次下标访问都会触发数组边界检查）：

```rust
let v = vec![1, 2, 3];
for i in &v {
    println!("{}", i);
}
```

也可以在迭代过程中，修改 `Vector` 中的元素：

```rust
let mut v = vec![1, 2, 3];
for i in &mut v {
    *i += 10
}
```



## 存储不同类型的元素

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
  
  数组 `v` 中存储了两种不同的 `ip` 地址，但是这两种都属于 `IpAddr` 枚举类型的成员，因此可以存储在数组中。

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
  
  为 `V4` 和 `V6` 都实现了特征 `IpAddr`，然后将它俩的实例用 `Box::new` 包裹后，存在了数组 `v` 中，需要注意的是，这里必需手动的指定类型：`Vec<Box<dyn IpAddr>>`，表示数组 `v` 存储的是特征 `IpAddr` 的对象，这样就实现了在数组中存储不同的类型。
  
  在实际使用场景中，特征对象数组要比枚举数组常见很多，主要原因在于**特征对象**非常灵活，而编译器对枚举的限制较多，且无法动态增加类型。


















