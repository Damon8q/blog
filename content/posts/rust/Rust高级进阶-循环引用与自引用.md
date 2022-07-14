---
title: "[Rust course]-19 高级进阶-循环引用与自引用"
date: 2022-07-05T16:54:00+08:00
lastmod: 2022-07-13T20:35:00+08:00
author: nange
draft: false
description: "Rust循环引用与自引用"

categories: ["programming"]
series: ["rust-course"]
series_weight: 19
tags: ["rust"]
---

在Rust中实现一个链表的难度是“地狱级”。链表在 Rust 中之所以这么难，完全是因为循环引用和自引用的问题引起的，这两个问题可以说综合了 Rust 的很多难点。本章将彻底解决这两个老大难问题。

## Weak与循环引用

Rust是一门内存安全的语言，并且不需要GC。但并不代表一定不会出现内存泄露。一个典型的内存泄露例子就是同时使用 `Rc<T>` 和 `RefCell<T>` 创建循环引用，最终这些引用的计数都无法被归零。

### Weak

`Weak` 非常类似于 `Rc`，但是与 `Rc` 持有所有权不同，`Weak` 不持有所有权，它仅仅保存一份指向数据的弱引用：如果你想要访问数据，需要通过 `Weak` 指针的 `upgrade` 方法实现，该方法返回一个类型为 `Option<Rc<T>>` 的值。

何为弱引用？就是**不保证引用关系依然存在**，如果不存在，就返回一个 `None`！

因为 `Weak` 引用不计入所有权，因此它**无法阻止所引用的内存值被释放掉**，而且 `Weak` 本身不对值的存在性做任何担保，引用的值还存在就返回 `Some`，不存在就返回 `None`。

使用方式简单总结下：**对于父子引用关系，可以让父节点通过 `Rc` 来引用子节点，然后让子节点通过 `Weak` 来引用父节点**。

#### Weak总结

`Weak` 通过 `use std::rc::Weak` 来引入，它具有以下特点:

- 可访问，但没有所有权，不增加引用计数，因此不会影响被引用值的释放回收
- 可由 `Rc<T>` 调用 `downgrade` 方法转换成 `Weak<T>`
- `Weak<T>` 可使用 `upgrade` 方法转换成 `Option<Rc<T>>`，如果资源已经被释放，则 `Option` 的值是 `None`
- 常用于解决循环引用的问题

一个简单的例子：

```rust
use std::rc::Rc;

fn main() {
    // 创建Rc，持有一个值5
    let five = Rc::new(5);

    // 通过Rc，创建一个Weak指针
    let weak_five = Rc::downgrade(&five);

    // Weak引用的资源依然存在，取到值5
    let strong_five: Option<Rc<_>> = weak_five.upgrade();
    assert_eq!(*strong_five.unwrap(), 5);

    // 手动释放资源`five`
    drop(five);

    // Weak引用的资源已不存在，因此返回None
    let strong_five: Option<Rc<_>> = weak_five.upgrade();
    assert_eq!(strong_five, None);
}
```

使用 `Weak` 让 Rust 本来就堪忧的代码可读性又下降了不少，但是可以解决循环引用了。

### 使用Weak解决循环引用

#### 工具间的故事

工具间里，每个工具都有其主人，且多个工具可以拥有一个主人；同时一个主人也可以拥有多个工具，在这种场景下，就很容易形成循环引用：

```rust
use std::rc::Rc;
use std::rc::Weak;
use std::cell::RefCell;

// 主人
struct Owner {
    name: String,
    gadgets: RefCell<Vec<Weak<Gadget>>>,
}

// 工具
struct Gadget {
    id: i32,
    owner: Rc<Owner>,
}

fn main() {
    // 创建一个 Owner
    // 需要注意，该 Owner 也拥有多个 `gadgets`
    let gadget_owner : Rc<Owner> = Rc::new(
        Owner {
            name: "Gadget Man".to_string(),
            gadgets: RefCell::new(Vec::new()),
        }
    );

    // 创建工具，同时与主人进行关联：创建两个 gadget，他们分别持有 gadget_owner 的一个引用。
    let gadget1 = Rc::new(Gadget{id: 1, owner: gadget_owner.clone()});
    let gadget2 = Rc::new(Gadget{id: 2, owner: gadget_owner.clone()});

    // 为主人更新它所拥有的工具
    // 因为之前使用了 `Rc`，现在必须要使用 `Weak`，否则就会循环引用
    gadget_owner.gadgets.borrow_mut().push(Rc::downgrade(&gadget1));
    gadget_owner.gadgets.borrow_mut().push(Rc::downgrade(&gadget2));

    // 遍历 gadget_owner 的 gadgets 字段
    for gadget_opt in gadget_owner.gadgets.borrow().iter() {

        // gadget_opt 是一个 Weak<Gadget> 。 因为 weak 指针不能保证他所引用的对象
        // 仍然存在。所以我们需要显式的调用 upgrade() 来通过其返回值(Option<_>)来判
        // 断其所指向的对象是否存在。
        // 当然，Option 为 None 的时候这个引用原对象就不存在了。
        let gadget = gadget_opt.upgrade().unwrap();
        println!("Gadget {} owned by {}", gadget.id, gadget.owner.name);
    }

    // 在 main 函数的最后，gadget_owner，gadget1 和 gadget2 都被销毁。
    // 具体是，因为这几个结构体之间没有了强引用（`Rc<T>`），所以，当他们销毁的时候。
    // 首先 gadget2 和 gadget1 被销毁。
    // 然后因为 gadget_owner 的引用数量为 0，所以这个对象可以被销毁了。
    // 循环引用问题也就避免了
}
```

### unsafe解决循环引用

除了使用 Rust 标准库提供的这些类型，还可以使用 `unsafe` 里的裸指针来解决这些棘手的问题。unsafe具体用法，将在后续章节再做学习，这里暂不深入。

虽然 `unsafe` 不安全，但是在各种库的代码中依然很常见用它来实现自引用结构，主要优点如下:

- 性能高，毕竟直接用裸指针操作
- 代码更简单更符合直觉: 对比下 `Option<Rc<RefCell<Node>>>`



## 结构体中的自引用

太复杂，没看懂，跳过。。 :) 


