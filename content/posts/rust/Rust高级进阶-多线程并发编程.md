---
title: "[Rust course]-20 高级进阶-多线程并发编程"
date: 2022-07-14T19:48:00+08:00
lastmod: 2022-08-02T16:40:00+08:00
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

### 在线程闭包中使用move

如果在一个线程中直接使用另一个线程的数据：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];
    
    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });
    handle.join().unwrap();
}
```

运行结果：

```rust
error[E0373]: closure may outlive the current function, but it borrows `v`, which is owned by the current function
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `v`
7 |         println!("Here's a vector: {:?}", v);
  |                                           - `v` is borrowed here
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:6:18
  |
6 |       let handle = thread::spawn(|| {
  |  __________________^
7 | |         println!("Here's a vector: {:?}", v);
8 | |     });
  | |______^
help: to force the closure to take ownership of `v` (and any other referenced variables), use the `move` keyword
  |
6 |     let handle = thread::spawn(move || {
  |                                ++++
```

问题在于 Rust 无法确定新的线程会活多久（多个线程的结束顺序并不是固定的），所以也无法确定新线程所引用的 `v` 是否在使用过程中一直合法。

使用 `move` 关键字拿走 `v` 的所有权即可：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();

    // 下面代码会报错borrow of moved value: `v`
    // println!("{:?}",v);
}
```

### 多线程的性能

* 创建线程的性能

  创建一个线程大概需要0.24毫秒，随着线程的变多，这个值会变得更大。因此创建线程是需要三思而行的。

* 创建多少线程合适

  由于CPU的核心数限制，当任务是CPU密集型时，创建超过核心数的线程是没有意义的。因为单个线程就能够让单个核心跑满，所以创建CPU核心数个线程是最好的。

  但当任务大部分时候都处于阻塞状态时，就可以考虑创建更多线程，比如对于网络IO操作，传统方式是一个线程用于监听请求，每来一个请求创建一个线程去处理，这种方式是OK的，但是有更好的方案，使用 `async/await` 的 `M:N` 并发模型，能够达到最大性能。

* 多线程的开销

  下面的代码是一个无锁实现(CAS)的 `Hashmap` 在多线程下的使用：

  ```rust
  for i in 0..num_threads {
      let ht = Arc::clone(&ht);
  
      let handle = thread::spawn(move || {
          for j in 0..adds_per_thread {
              let key = thread_rng().gen::<u32>();
              let value = thread_rng().gen::<u32>();
              ht.set_item(key, value);
          }
      });
  
      handles.push(handle);
  }
  
  for handle in handles {
      handle.join().unwrap();
  }
  ```

  既然是无锁实现了，那么锁的开销应该几乎没有，性能会随着线程数的增加接近线性增长，但是真的是这样吗？该代码在 `48` 核机器上的运行结果表明：线程数达到16时，继续增加反而会造成性能下降。

  大概的原因有：

  * 虽然是无锁，但是内部是 CAS 实现，大量线程的同时访问，会让 CAS 重试次数大幅增加
  * 线程过多时，CPU 缓存的命中率会显著下降，同时多个线程竞争一个 CPU Cache-line 的情况也会经常发生
  * 大量读写可能会让内存带宽也成为瓶颈
  * 读和写不一样，无锁数据结构的读往往可以很好地线性增长，但是写不行，因为写竞争太大

总之，多线程的开销往往是在锁、数据竞争、缓存失效上，这些限制了现代化软件系统随着 CPU 核心的增多性能也线性增加的野心。



### 线程屏障

在 Rust 中，可以使用 `Barrier` 让多个线程都执行到某个点后，才继续一起往后执行：

```rust
use std::sync::{Arc, Barrier};
use std::thread;

fn main() {
    let mut handles = Vec::with_capacity(6);
    let barrier = Arc::new(Barrier::new(6));
    
    for _ in 0..6 {
        let b = barrier.clone();
        handles.push(thread::spawn(move || {
            println!("before wait");
            b.wait();
            println!("after wait");
        }));
    }
    
    for handle in handles {
        handle.join.unwrap();
    }
}

```

上面代码，我们在线程打印出 `before wait` 后增加了一个屏障，目的就是等所有的线程都打印出**before wait**后，各个线程再继续执行：

```rust
before wait
before wait
before wait
before wait
before wait
before wait
after wait
after wait
after wait
after wait
after wait
after wait
```



### 线程局部变量(Thread Local Variable)

对于多线程编程，线程局部变量在一些场景下非常有用， Rust 通过标准库和三方库对此进行了支持。

* **标准库thread_local**

  使用 `thread_local` 宏可以初始化线程局部变量，然后在线程内部使用该变量的 `with` 方法获取变量值：

  ```rust
  use std::cell::RefCell;
  use std::thread;
  
  thread_local!(static FOO: RefCell<u32> = RefCell::new(1));
  
  FOO.with(|f| {
      assert_eq!(*f.borrow(), 1);
      *f.borrow_mut() = 2;
  });
  
  // 每个线程开始时都会拿到线程局部变量的FOO的初始值
  let t = thread::spawn(move|| {
      FOO.with(|f| {
          assert_eq!(*f.borrow(), 1);
          *f.borrow_mut() = 3;
      });
  });
  
  // 等待线程完成
  t.join().unwrap();
  
  // 尽管子线程中修改为了3，我们在这里依然拥有main线程中的局部值：2
  FOO.with(|f| {
      assert_eq!(*f.borrow(), 2);
  });
  ```

  上面代码中，`FOO` 即是我们创建的**线程局部变量**，每个新的线程访问它时，都会使用它的初始值作为开始，各个线程中的 `FOO` 值彼此互不干扰。注意 `FOO` 使用 `static` 声明为生命周期为 `'static` 的静态变量。

* 第三方库 thread_local

  使用`thread_local`库，它允许每个线程持有值的独立拷贝：

  ```rust
  use thread_local::ThreadLocal;
  use std::sync::Arc;
  use std::cell::Cell;
  use std::thread;
  
  let tls = Arc::new(ThreadLocal::new());
  
  // 创建多个线程
  for _ in 0..5 {
      let tls2 = tls.clone();
      thread::spawn(move || {
          // 将计数器加1
          let cell = tls2.get_or(|| Cell::new(0));
          cell.set(cell.get() + 1);
      }).join().unwrap();
  }
  
  // 一旦所有子线程结束，收集它们的线程局部变量中的计数器值，然后进行求和
  let tls = Arc::try_unwrap(tls).unwrap();
  let total = tls.into_iter().fold(0, |x, y| x + y.get());
  
  // 和为5
  assert_eq!(total, 5);
  ```

  该库不仅仅使用了值的拷贝，而且还能自动把多个拷贝汇总到一个迭代器中，最后进行求和，非常好用。



### 使用条件控制线程的挂起和执行

条件变量（Condition Variables）经常和`Mutex`一起使用，可以让线程挂起，直到某个条件发生后再继续执行：

```rust
use std::sync::Arc;
use std::sync::Condvar;
use std::sync::Mutex;
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = pair.clone();

    thread::spawn(move || {
        let &(ref lock, ref cvar) = &*pair2;
        let mut started = lock.lock().unwrap();
        println!("changing started...");

        *started = true;
        cvar.notify_one();
    });

    let &(ref lock, ref cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    while !*started {
        started = cvar.wait(started).unwrap();
    }

    println!("started changed...");
}
```

上述代码流程如下：

1. `main` 线程首先进入 `while` 循环，调用 `wait` 方法挂起等待子线程的通知，并释放了锁 `started`
2. 子线程获取到锁，并将其修改为 `true`，然后调用条件变量的 `notify_one` 方法来通知主线程继续执行

### 只被调用一次的函数

有时，我们会需要某个函数在多线程环境下只被调用一次，例如初始化全局变量，无论是哪个线程先调用函数来初始化，都会保证全局变量只会被初始化一次，随后的其它线程调用就会忽略该函数：

```rust
use std::sync::Once;
use std::thread;

static mut VAL: usize = 0;
static INIT: Once = Once::new();

fn main() {
    let handle1 = thread::spawn(move || {
        INIT.call_once(|| unsafe {
            VAL = 1;
        });
    });
    let handle2 = thread::spawn(move || {
        INIT.call_once(|| unsafe {
            VAL = 2;
        });
    });

    handle1.join().unwrap();
    handle2.join().unwrap();

    println!("{}", unsafe { VAL });
}
```

代码运行的结果取决于哪个线程先调用 `INIT.call_once` （虽然代码具有先后顺序，但是线程的初始化顺序并无法被保证！因为线程初始化是异步的，且耗时较久），若 `handle1` 先，则输出 `1`，否则输出 `2`。

**call_once 方法：** 执行初始化过程一次，并且只执行一次。如果当前有另一个初始化过程正在运行，线程将阻止该方法被调用。当这个函数返回时，保证初始化已经运行并完成，它还保证所执行的任何内存写入都能被其他线程在这时可靠地观察到。







