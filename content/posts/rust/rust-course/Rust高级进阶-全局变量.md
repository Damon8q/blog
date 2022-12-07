---
title: "[Rust course]-21 高级进阶-全局变量"
date: 2022-10-08T19:30:00+08:00
lastmod: 2022-10-09T15:40:00+08:00
author: nange
draft: false
description: "Rust全局变量"

categories: ["programming"]
series: ["rust-course"]
series_weight: 21
tags: ["rust"]
---

全局变量分为编译期初始化和运行期初始化两类，它们的生命周期都是`'static'`，也就是和程序运行一样久。

## 编译期初始化

### 静态常量

常量，顾名思义它是不可变的，很适合用作静态配置：

```rust
const MAX_ID: usize =  usize::MAX / 2;

fn main() {
   println!("用户ID允许的最大值是{}",MAX_ID);
}
```

**常量与普通变量的区别**:

* 关键字是`const`而不是`let`
* 定义常量必须指明类型（如 i32）不能省略
* 定义常量时变量的命名规则一般是全部大写
* 常量可以在任意作用域进行定义，其生命周期贯穿整个程序的生命周期。编译时编译器会尽可能将其内联到代码中，所以在不同地方对同一常量的引用并不能保证引用到相同的内存地址
* 常量必须是在编译期就能计算出的值，如果需要在运行时才能得出结果的值比如函数，则不能赋值给常量表达式
* 常量不同于变量，不允许出现重复的定义

### 静态变量

静态变量允许声明一个全局的变量，常用于全局数据统计，例如我们希望用一个变量来统计程序当前的总请求数：

```rust
static mut REQUEST_RECV: usize = 0;
fn main() {
   unsafe {
        REQUEST_RECV += 1;
        assert_eq!(REQUEST_RECV, 1);
   }
}
```

Rust 要求必须使用`unsafe`语句块才能访问和修改`static`变量，因为在多线程中同时去修改，是不安全的。只有在同一线程内或者不在乎数据的准确性时，才应该使用全局静态变量。

和常量相同，定义静态变量必须赋值为在编译期就可以计算出的值。

**静态变量和常量的区别**：

- 静态变量不会被内联，在整个程序中，静态变量只有一个实例，所有的引用都会指向同一个地址
- 存储在静态变量中的值必须要实现 Sync trait

### 原子类型

想要全局计数器、状态控制等功能，又想要线程安全的实现，可以使用原子类型。

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
static REQUEST_RECV: AtomicUsize  = AtomicUsize::new(0);
fn main() {
    for _ in 0..100 {
        REQUEST_RECV.fetch_add(1, Ordering::Relaxed);
    }

    println!("当前用户请求数{:?}",REQUEST_RECV);
}
```

### 示例：全局ID生成器

```rust
use std::sync::atomic::{Ordering, AtomicUsize};

struct Factory{
    factory_id: usize,
}

static GLOBAL_ID_COUNTER: AtomicUsize = AtomicUsize::new(0);
const MAX_ID: usize = usize::MAX / 2;

fn generate_id()->usize{
    // 检查两次溢出，否则直接加一可能导致溢出
    let current_val = GLOBAL_ID_COUNTER.load(Ordering::Relaxed);
    if current_val > MAX_ID{
        panic!("Factory ids overflowed");
    }
    let next_id = GLOBAL_ID_COUNTER.fetch_add(1, Ordering::Relaxed);
    if next_id > MAX_ID {
        panic!("Factory ids overflowed");
    }
    next_id
}

impl Factory{
    fn new()->Self{
        Self{
            factory_id:generate_id()
        }
    }
}
```



## 运行期初始化

上面静态初始化最大的问题就是**不能用函数进行静态初始化。**例如像下面这样声明一个全局的Mutex锁会报错：

```rust
use std::sync::Mutex;
static NAMES: Mutex<String> = Mutex::new(String::from("Sunface, Jack, Allen"));

fn main() {
    let v = NAMES.lock().unwrap();
    println!("{}",v);
}
```

运行后报错如下：

```rust
error[E0015]: calls in statics are limited to constant functions, tuple structs and tuple variants
 --> src/main.rs:3:42
  |
3 | static NAMES: Mutex<String> = Mutex::new(String::from("sunface"));
```

但Rust在定义一个变量时，又必须对其初始化。这种情况可以使用`lazy_static`这个包。

### lazy_static

lazy_static是一个很强大的宏，它是在运行期初始化静态变量。如：

```rust
use std::sync::Mutex;
use lazy_static::lazy_static;
lazy_static! {
    static ref NAMES: Mutex<String> = Mutex::new(String::from("Sunface, Jack, Allen"));
}

fn main() {
    let mut v = NAMES.lock().unwrap();
    v.push_str(", Myth");
    println!("{}",v);
}
```

lazy_static底层使用到了std::sync::Once并发原语，每次访问该变量时，都会执行一次原子指令用于确认静态变量初始化是否完成。

`lazy_static`宏，匹配的是`static ref`，所以定义的静态变量都是不可变引用。上面还能`push_str`是因为`Mutex`是一个智能指针，其本身就实现了内部可变性。

还有一个常见的使用运行期全局静态变量的地方是：**一个全局的动态配置，它在程序开始后，才加载数据进行初始化，最终可以让各个线程直接访问使用**。如：

```rust
use lazy_static::lazy_static;
use std::collections::HashMap;

lazy_static! {
    static ref HASHMAP: HashMap<u32, &'static str> = {
        let mut m = HashMap::new();
        m.insert(0, "foo");
        m.insert(1, "bar");
        m.insert(2, "baz");
        m
    };
}

fn main() {
    // 首次访问`HASHMAP`的同时对其进行初始化
    println!("The entry for `0` is \"{}\".", HASHMAP.get(&0).unwrap());

    // 后续的访问仅仅获取值，再不会进行任何初始化操作
    println!("The entry for `1` is \"{}\".", HASHMAP.get(&1).unwrap());
    
    // 如果执行插入操作，则编译报错，HashMap是只读的
    // HASHMAP.insert(3, "hello");
}
```

需要注意的是，`lazy_static`直到运行到`main`中的第一行使用到`HASHMAP`的时候，才进行初始化，这就是lazy的含义。


