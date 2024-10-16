---
title: 并发、线程、通信
pubDatetime: 2024-10-16T15:35:00+08:00
author: shushunie
slug: rust-l4-08
description: ""
---

## 并发和并行

- **并发(Concurrent)** 是多个队列使用同一个咖啡机，然后两个队列轮换着使用（未必是 1:1 轮换，也可能是其它轮换规则），最终每个人都能接到咖啡

- **并行(Parallel)** 是每个队列都拥有一个咖啡机，最终也是每个人都能接到咖啡，但是效率更高，因为同时可以有两个人在接咖啡

#### CPU 并发模型

当代 cpu 都拥有多个核心，多核心对的任务处理方式结合了并行和并发。简单来讲，把各种任务简单分成多个队列，每个队列都交给一个 CPU 核心去执行（并行），当某个 CPU 核心没有任务时，它还能处理其他队列的任务（并发）

#### 编程语言的并发模型

由于各编程语言的特性差异，不同语言对于线程的实现可能大相径庭

- **1:1 线程模型：**直接调用该 API 来创建线程，最终程序内的线程数和该程序占用的操作系统线程数相等

- **M:N 线程模型：**语言在内部实现了自己的线程模型（绿色线程、协程），程序内部的 M 个线程最后会以某种映射方式使用 N 个操作系统线程去运行

- **Actor 模型：**基于消息传递进行并发

## 使用线程

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

    handle.join().unwrap();

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

使用 `thread::spawn` 可以创建线程

`thread::sleep` 会让当前线程休眠指定的时间，操作系统会将 CPU 时间分配给其他就绪状态的线程

`handle.join` 可以让当前线程阻塞，直到它等待的子线程的结束。在上面代码中，由于 `main` 线程会被阻塞，因此它直到子线程结束后才会输出自己的 `1..5`

线程往往是轮流执行的，但是这一点无法被保证。调度的方式往往取决于使用的操作系统，所以不要依赖线程的执行顺序

#### 线程中的所有权

Rust 无法确定新的线程会活多久（多个线程的结束顺序并不是固定的），所以也无法确定新线程所引用的变量是否在使用过程中一直合法。例如当主线程执行完，变量内存释放掉时，新的线程很可能还没有结束甚至还没有被创建成功

针对这种情况，需要将所有权在线程之间进行转移。这一点可以通过闭包的按值捕获来实现

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

## 多线程同步

在多线程编程中，同步性极其的重要，当你需要同时访问一个资源、控制不同线程的执行次序时，都需要使用到同步性。例如我们可以通过消息传递来控制不同线程间的执行次序，还可以使用共享内存来实现多个线程同时且安全地去访问一个资源

#### 消息传递

Rust 在标准库里提供了消息通道(`channel`)，一个通道支持多个发送者和接收者

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
    println!("receive {:?}", rx.try_recv());
}
```

`mpsc`是 multiple producer, single consumer 的缩写，代表了该通道支持多个发送者，但是只支持唯一的接收者

可以创建异步通道 `channel` 或同步通道 `sync_channel`。同步通道发送消息是阻塞的，只有在消息被接收后才解除阻塞

`tx`,`rx`对应发送者和接收者，它们的类型由编译器自动推导: `tx.send(1)`发送了整数，因此它们分别是`mpsc::Sender<i32>`和`mpsc::Receiver<i32>`类型。由于内部是泛型实现，一旦类型被推导确定，该通道就只能传递对应类型的值, 例如此例中非`i32`类型的值将导致编译错误

接收消息的操作`rx.recv()`会阻塞当前线程，直到读取到值，或者通道被关闭。`try_recv`尝试接收一次消息，该方法并不会阻塞线程，当通道中没有消息时，它会立刻返回一个错误

`send`方法返回一个`Result<T,E>`，在代码中我们仅仅使用`unwrap`进行了快速处理，但在实际项目中你需要对错误进行进一步的处理

#### 共享内存

消息传递类似一个单所有权的系统：一个值同时只能有一个所有者，如果另一个线程需要该值的所有权，需要将所有权通过消息传递进行转移。而共享内存类似于一个多所有权的系统：多个线程可以访问同一个值

多个线程可以访问同一个值往往是不安全的，这时候需要使用`Mutex`让多个线程并发的访问同一个值变成了排队访问：同一时间，只允许一个线程`A`访问该值，其它线程需要等待`A`访问完成后才能继续

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![]; // 创建向量用于存储线程句柄

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap(); // 等待所有线程执行完毕
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

上述代码使用了 `Arc` 使 counter 有多个所有者而非 `Rc`，这是由于 `Rc` 仅适用于单线程

#### Send 和 Sync 特征简介

`Rc<T>` 的 `Send` 和 `Sync` 特征被特地移除了实现，而 `Arc<T>` 则相反，实现了`Sync + Send`。实际上它们只是标记特征(marker trait，该特征未定义任何行为，因此非常适合用于标记）

在 Rust 中，几乎所有类型都默认实现了`Send`和`Sync`，而且由于这两个特征都是可自动派生的特征(通过`derive`派生)，意味着一个复合类型(例如结构体), 只要它内部的所有成员都实现了`Send`或者`Sync`，那么它就自动实现了`Send`或`Sync`
