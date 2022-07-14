---
title: "[Rust course]-20 高级进阶-多线程并发编程"
date: 2022-07-14T19:48:00+08:00
lastmod: 2022-07-14T19:48:00+08:00
author: nange
draft: false
description: "Rust多线程并发编程"

categories: ["programming"]
series: ["rust-course"]
series_weight: 20
tags: ["rust"]
---

## 使用多线程

### 多线程编程的风险

由于多线程的代码是同时运行的，无法保证线程间的执行顺序，这会导致一些问题：

- 竞态条件(race conditions)，多个线程以非一致性的顺序同时访问数据资源
- 死锁(deadlocks)，两个线程都想使用某个资源，但是又都在等待对方释放资源后才能使用，结果最终都无法继续执行
- 一些因为多线程导致的很隐晦的 BUG，难以复现和解决

虽然 Rust 已经通过各种机制减少了上述情况的发生，但是依然无法完全避免上述情况，因此我们在编程时需要格外的小心，后面章节也会列出多线程编程时常见的陷阱。

### 创建线程

使用 `thread::spawn` 可以创建线程：

```rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

几点值得注意：

- 线程内部的代码使用闭包来执行
- `main` 线程一旦结束，程序就立刻结束，因此需要保持它的存活，直到其它子线程完成自己的任务

输出：

```rust
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 2 from the main thread!
hi number 3 from the spawned thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
```

多运行几次，会发现好像每次输出会不太一样，因为线程调度的方式往往取决于你使用的操作系统。所以，**千万不要依赖线程的执行顺序**。

### 等待子线程的结束

上面的代码，可能子线程还没执行完成，主线程就退出了，造成整个进程退出，更好的方式是主线程能够等到子线程都干完活后再退出。

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..5 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    handle.join().unwrap();

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

调用`handle.join`，可以阻塞当前线程，直到它等待的子线程结束。因此上面的代码输出：

```rust
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
```

如果将 `handle.join` 放置在 `main` 线程中的 `for` 循环后面，那就是另外一个结果：两个线程交替输出。













