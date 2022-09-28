---
title: "[Rust course]-20 高级进阶-多线程并发编程"
date: 2022-07-14T19:48:00+08:00
lastmod: 2022-09-28T18:14:00+08:00
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



## 线程同步：消息传递

### 多发送者，单接受者

标准库提供了通道`std::sync::mpsc`，其中`mpsc`是*multiple producer, single consumer*的缩写，看一个例子：

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // 创建一个消息通道, 返回一个元组：(发送者，接收者)
    let (tx, rx) = mpsc::channel();

    // 创建线程，并发送消息
    thread::spawn(move || {
        // 发送一个数字1, send方法返回Result<T,E>，通过unwrap进行快速错误处理
        tx.send(1).unwrap();

        // 下面代码将报错，因为编译器自动推导出通道传递的值是i32类型，那么Option<i32>类型将产生不匹配错误
        // tx.send(Some(1)).unwrap()
    });
	
    // 在主线程中接收子线程发送的消息并输出
    println!("receive {}", rx.recv().unwrap());
}
```

在注释中提到`send`方法返回一个`Result<T,E>`，说明它有可能返回一个错误，例如接收者被`drop`导致了发送的值不会被任何人接收，此时继续发送毫无意义，因此返回一个错误最为合适，在代码中我们仅仅使用`unwrap`进行了快速处理，但在实际项目中需要对错误进行进一步的处理。

同样的，对于`recv`方法来说，当发送者关闭时，它也会接收到一个错误，用于说明不会再有任何值被发送过来。

注意：`send`方法不会阻塞当前线程，即使接受者还未开始接收消息。而`recv`会阻塞当前线程，知道读取到值，或者通道关闭。

### 不阻塞的 try_recv 方法

除了上述`recv`方法，还可以使用`try_recv`尝试接收一次消息，该方法并**不会阻塞线程**，当通道中没有消息时，它会立刻返回一个错误：

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(1).unwrap();
    });

    println!("receive {:?}", rx.try_recv());
}
```

由于子线程的创建需要时间，因此`println!`和`try_recv`方法会先执行，而此时子线程的**消息还未被发出**。此次读取最终会报错:

```rust
receive Err(Empty)
```

### 传输具有所有权的数据

使用通道来传输数据，一样要遵循 Rust 的所有权规则：

- 若值的类型实现了`Copy`特征，则直接复制一份该值，然后传输过去，例如之前的`i32`类型
- 若值没有实现`Copy`，则它的所有权会被转移给接收端，在发送端继续使用该值将报错

看看第二种情况:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let s = String::from("我，飞走咯!");
        tx.send(s).unwrap();
        println!("val is {}", s);
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

编译报错：

```rust
error[E0382]: borrow of moved value: `s`
  --> src/main.rs:10:31
   |
8  |         let s = String::from("我，飞走咯!");
   |             - move occurs because `s` has type `String`, which does not implement the `Copy` trait // 所有权被转移，由于`String`没有实现`Copy`特征
9  |         tx.send(s).unwrap();
   |                 - value moved here // 所有权被转移走
10 |         println!("val is {}", s);
   |                               ^ value borrowed here after move // 所有权被转移后，依然对s进行了借用
```

### 使用 for 循环接收

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

当子线程运行完成时，发送者`tx`会随之被`drop`，此时`for`循环将被终止，最终`main`线程成功结束。

#### 使用多发送者

由于子线程会拿走发送者的所有权，因此我们必须对发送者进行克隆，然后让每个线程拿走它的一份拷贝:

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx1 = tx.clone();
    thread::spawn(move || {
        tx.send(String::from("hi from raw tx")).unwrap();
    });

    thread::spawn(move || {
        tx1.send(String::from("hi from cloned tx")).unwrap();
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

有几点需要注意:

- 需要所有的发送者都被`drop`掉后，接收者`rx`才会收到错误，进而跳出`for`循环，最终结束主线程
- 这里虽然用了`clone`但是并不会影响性能，因为它并不在热点代码路径中，仅仅会被执行一次
- 由于两个子线程谁先创建完成是未知的，因此哪条消息先发送也是未知的，最终主线程的输出顺序也不确定

### 消息顺序

上述第三点的消息顺序仅仅是因为线程创建引起的，并不代表通道中的消息是无序的，对于通道而言，消息的发送顺序和接收顺序是一致的，满足`FIFO`原则(先进先出)。

### 同步和异步通道

Rust 标准库的`mpsc`通道其实分为两种类型：同步和异步。

#### 异步通道

之前我们使用的都是异步通道：无论接收者是否正在接收消息，消息发送者在发送消息时都不会阻塞。不再举例。

#### 同步通道

与异步通道相反，同步通道**发送消息是阻塞的，只有在消息被接收后才解除阻塞**，例如：

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;
fn main() {
    let (tx, rx)= mpsc::sync_channel(0);

    let handle = thread::spawn(move || {
        println!("发送之前");
        tx.send(1).unwrap();
        println!("发送之后");
    });

    println!("睡眠之前");
    thread::sleep(Duration::from_secs(3));
    println!("睡眠之后");

    println!("receive {}", rx.recv().unwrap());
    handle.join().unwrap();
}
```

运行后输出如下：

```rust
睡眠之前
发送之前
//···睡眠3秒
睡眠之后
receive 1
发送之后
```

#### 消息缓存

上面的代码中，传递了一个参数`0`: `mpsc::sync_channel(0);`，这里的0和go语言中，`chan`的长度有类似的作用。 当通道还未满时，发送就会是异步的，通道满后，再发送就会变为同步。

使用异步消息虽然能非常高效且不会造成发送线程的阻塞，但是存在消息未及时消费，最终内存过大的问题。在实际项目中，可以考虑使用一个带缓冲值的同步通道来避免这种风险。

### 关闭通道

之前我们数次提到了通道关闭，并且提到了当通道关闭后，发送消息或接收消息将会报错。那么如何关闭通道呢？ 很简单：**所有发送者被`drop`或者所有接收者被`drop`后，通道会自动关闭**。

这件事是在编译期实现的，完全没有运行期性能损耗！

### 传输多种类型的数据

之前提到过，一个消息通道只能传输一种类型的数据，如果你想要传输多种类型的数据，可以为每个类型创建一个通道，你也可以使用枚举类型来实现：

```rust
use std::sync::mpsc::{self, Receiver, Sender};

enum Fruit {
    Apple(u8),
    Orange(String)
}

fn main() {
    let (tx, rx): (Sender<Fruit>, Receiver<Fruit>) = mpsc::channel();

    tx.send(Fruit::Orange("sweet".to_string())).unwrap();
    tx.send(Fruit::Apple(2)).unwrap();

    for _ in 0..2 {
        match rx.recv().unwrap() {
            Fruit::Apple(count) => println!("received {} apples", count),
            Fruit::Orange(flavor) => println!("received {} oranges", flavor),
        }
    }
}
```

如上所示，枚举类型还能让我们带上想要传输的数据，但是有一点需要注意，Rust 会按照枚举中占用内存最大的那个成员进行内存对齐，这意味着就算你传输的是枚举中占用内存最小的成员，它占用的内存依然和最大的成员相同, 因此会造成内存上的浪费。

### 新手容易遇到的坑

```rust
use std::sync::mpsc;
fn main() {

    use std::thread;

    let (send, recv) = mpsc::channel();
    let num_threads = 3;
    for i in 0..num_threads {
        let thread_send = send.clone();
        thread::spawn(move || {
            thread_send.send(i).unwrap();
            println!("thread {:?} finished", i);
        });
    }

    // 在这里drop send...

    for x in recv {
        println!("Got: {}", x);
    }
    println!("finished iterating");
}
```

之前提到，通道关闭的两个条件：发送者全部`drop`或接收者被`drop`，要结束`for`循环显然是要求发送者全部`drop`，但是由于`send`自身没有被`drop`，会导致该循环永远无法结束，最终主线程会一直阻塞。

解决办法很简单，`drop`掉`send`即可：在代码中的注释下面添加一行`drop(send);`。

### mpmc 更好的性能

如果需要 mpmc(多发送者，多接收者)或者需要更高的性能，可以考虑第三方库:

- [**crossbeam-channel**](https://github.com/crossbeam-rs/crossbeam/tree/master/crossbeam-channel), 老牌强库，功能较全，性能较强，之前是独立的库，但是后面合并到了`crossbeam`主仓库中
- [**flume**](https://github.com/zesterer/flume), 官方给出的性能数据某些场景要比 crossbeam 更好些



## 线程同步：锁、Condvar和信号量

### 互斥锁 Mutex

同一时间只能一个线程访问。

#### 单线程中使用 Mutex

```rust
use std::sync::Mutex;
fn main() {
    // 使用`Mutex`结构体的关联函数创建新的互斥锁实例
    let m = Mutex::new(5);

    {
        // 获取锁，然后deref为`m`的引用
        // lock返回的是Result
        let mut num = m.lock().unwrap();
        *num = 6;
        // 锁自动被drop
    }

    println!("m = {:?}", m);
}
```

数据被`Mutex`所拥有（包裹封装），要访问内部的数据，需要使用方法`m.lock()`向`m`申请一个锁, 该方法会**阻塞当前线程，直到获取到锁**。

**`m.lock()`方法也有可能报错**，例如当前正在持有锁的线程`panic`了。在这种情况下，其它线程不可能再获得锁，因此`lock`方法会返回一个错误。

`m.lock()`返回一个智能指针`MutexGuard<T>`:

- 它实现了`Deref`特征，会被自动解引用后获得一个引用类型，该引用指向`Mutex`内部的数据
- 它还实现了`Drop`特征，在超出作用域后，自动释放锁，以便其它线程能继续获取锁

如果不注意作用域的使用，或者没有drop，继续调用`m.lock`则会发生死锁，如：

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    let mut num = m.lock().unwrap();
    *num = 6;
    // 锁还没有被 drop 就尝试申请下一个锁，导致主线程阻塞
    // drop(num); // 手动 drop num ，可以让 num1 申请到下个锁
    let mut num1 = m.lock().unwrap();
    *num1 = 7;
    // drop(num1); // 手动 drop num1 ，观察打印结果的不同

    println!("m = {:?}", m); // 如果没有drop(num1)，则打印不出具体内容，因为被加锁了
}
```

#### 多线程中使用 Mutex

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;

fn main() {
    // Rc对象是用于单线程使用，而Arc是用于多线程使用
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        // 每一个线程单独一个counter，因为传递给每个线程是需要独立的所有权的
        let counter = Arc::clone(&counter);

        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {:?}", counter.lock().unwrap());
}
```

简单总结：`Rc<T>/RefCell<T>`用于单线程内部可变性， `Arc<T>/Mutex<T>`用于多线程内部可变性。

另外当一个操作试图锁住两个资源，然后两个线程各自获取其中一个锁，并试图获取另一个锁时，就容易造成死锁。

#### try_lock

相比于`lock`方法，`try_lock`不会造成方法的阻塞，如果这次调用无法获取锁，则返回一个Err，相反能正常获取锁，就返回一个`MutexGuard`（智能指针）。

> 一个有趣的命名规则：在 Rust 标准库中，使用`try_xxx`都会尝试进行一次操作，如果无法完成，就立即返回，不会发生阻塞。例如消息传递章节中的`try_recv`以及本章节中的`try_lock`

### 读写锁 RwLock

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);

    // 同时允许多个读
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // 读锁在这里被drop，释放

    // 同时只允许一个写
    {
        let mut w = lock.write().unwrap();
        *w += 1;
        assert_eq!(*w, 6);

        // 写锁没释放，发生读，根据文档说，单线程下可能会panic
        // 但实际测试，并没有panic，而是发生死锁
        // let r1 = lock.read();
        // println!("{:?}", r1);
    } // 写锁在这里被drop，释放
}
```

多线程下的使用：

```rust
use std::sync::Arc;
use std::sync::RwLock;
use std::thread;
use std::time::Duration;

fn main() {
    let lock = Arc::new(RwLock::new(5));
    let c_lock = Arc::clone(&lock);

    let num = lock.read().unwrap();
    assert_eq!(*num, 5);

    let handle = thread::spawn(move || {
        let mut num = c_lock.write().unwrap();
        *num += 1;
    });

    // 注释掉下面这两行的话，会出现死锁。
    // 因为获取到读锁后，一直没有释放，线程中的写锁将永远处于等待状态
    drop(num);
    thread::sleep(Duration::from_millis(1000));

    handle.join().unwrap();
}
```

### 用条件变量(Condvar)控制线程的同步

`Mutex`用于解决资源安全访问的问题，但是还需要一个手段来解决资源访问顺序的问题。而 Rust 为我们提供了条件变量(Condition Variables)，它经常和`Mutex`一起使用，可以让线程挂起，直到某个条件发生后再继续执行

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = Arc::clone(&pair);

    thread::spawn(move || {
        let (lock, cvar) = &*pair2;

        thread::sleep(time::Duration::from_secs(4));

        println!("线程内获取锁");
        let mut started = lock.lock().unwrap();
        *started = true;
		cvar.notify_one();
        println!("线程内锁被释放");
    });

    // Wait for the thread to start up.
    let (lock, cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    // As long as the value inside the `Mutex<bool>` is `false`, we wait.
    while !*started {
        println!("调用 wait 之前");
        started = cvar.wait(started).unwrap();
        println!("接收到 wait 已触发");
    }
}
```



### 信号量 Semaphore

在多线程中，另一个重要的概念就是信号量，使用它可以让我们精准的控制当前正在运行的任务最大数量。Rust 在标准库中有提供一个[信号量实现](https://doc.rust-lang.org/1.8.0/std/sync/struct.Semaphore.html), 但是由于各种原因这个库现在已经不再推荐使用了，推荐使用`tokio`中提供的`Semaphore`实现: [`tokio::sync::Semaphore`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/semaphore.rs)。

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3));
    let mut join_handles = Vec::new();

    for _ in 0..5 {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        join_handles.push(tokio::spawn(async move {
            //
            // 在这里执行任务...
            //
            drop(permit);
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

上面代码创建了一个容量为 3 的信号量，当正在执行的任务超过 3 时，剩下的任务需要等待正在执行任务完成并减少信号量后到 3 以内时，才能继续执行。

这里的关键在于：信号量的申请和归还，使用前需要申请信号量，如果容量满了，就需要等待；使用后需要释放信号量，以便其它等待者可以继续。















