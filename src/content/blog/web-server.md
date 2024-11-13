---
title: 实现一个 web 服务器
pubDatetime: 2024-11-13T17:05:00+08:00
author: shushunie
slug: web-server
description: "使用 Rust 实现一个简单的 web 服务器"
---

## 基础功能

#### 监听 TCP 连接

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    for stream in listener.incoming() {
        let mut stream = stream.unwrap();

        let request_vec = handle_stream(&stream);
        handle_response(request_vec, stream);
    }
}
```

`TcpListener::bind` 监听本地 7878 端口，返回一个返回一个 `TcpListener` 实例

`incoming` 会返回一个迭代器，它每一次迭代都会返回一个新的连接 `stream`。客户端的请求数据可以从 `stream` 中读取

#### 处理请求

```rust
fn handle_stream(stream: &TcpStream) -> Vec<String> {
    let buf_reader = BufReader::new(stream);

    buf_reader
        .lines()
        .map(|line| line.unwrap())
        .take_while(|line| !line.is_empty())
        .collect()
}
```

`TcpStream` 本身不提供按行读取的功能，需要使用 `BufReader` 处理请求数据

`lines` 是 `BufReader` 提供的一个方法，返回一个按行读取的迭代器，每次迭代读取一行数据

`map` 遍历每一行的数据，保留 `OK` 值，忽略错误

`take_while` 再次遍历各行数据，遇到第一个空行时停止

`collect` 方法将符合条件的迭代器中的每个 `String` 元素收集成一个 `Vec`

#### 生成响应

```rust
fn create_response(status_line: &str, contents: &str) -> String {
    format!("{status_line}\r\n{{contents.len()}}\r\n\r\n{contents}")
}
```

主要使用 `forma!` 宏格式化响应数据

`status_line` 表示 HTTP 响应状态行

`\r\n` 表示换行符，`\r\n\r\n` 是连续换两行，会形成一个空行

`contents.len()` 返回正文内容的字节长度，对应 `Content-Length` 字段

`contents` 正文内容，包含实际的 HTML 或其他内容

假设输入如下：

```rust
let status_line = "HTTP/1.1 200 OK";
let contents = "<html><body>Hello, world!</body></html>";
```

生成的响应为：

```rust
HTTP/1.1 200 OK
29

<html><body>Hello, world!</body></html>
```

#### 返回数据

```rust
fn handle_response(request_vec: Vec<String>, mut stream: TcpStream) {
    let response = match request_vec[0].as_str() {
        "GET / HTTP/1.1" => {
            let hello_html_string = fs::read_to_string("hello.html").unwrap();
            create_response("HTTP/1.1 200 OK", &hello_html_string)
        }
        _ => {
            let err_html_string = fs::read_to_string("err.html").unwrap();
            create_response("HTTP/1.1 404 NOT FOUND", &err_html_string)
        }
    };

    stream.write_all(response.as_bytes()).unwrap();
}
```

`request_vec[0]` 是请求的第一行（通常是 HTTP 请求行，例如 GET / HTTP/1.1）。`as_str()` 方法将其转换为字符串切片用于模式匹配

使用 `match` 匹配不同的请求行，返回对应数据。这里简单分为 get 请求和其他情况

`fs::read_to_string("hello.html")` 用于读取文件 `hello.html` 的内容并转换为 `String`。读取文件内容后调用 `create_response` 函数生成返回数据

最后 `stream.write_all` 将 `response` 的字节流写入到客户端的 TCP 流中，发送 HTTP 响应

#### 结果验证

浏览器访问 `127.0.0.1:7878` 能够看到 hello 页面，访问其他该地址下的其他路由可以看到错误页面

---

[image](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/51c272dc-55af-453d-a95f-c85b15dc773a)

---

[image](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/654ee0b8-1fe3-4ce3-88a9-5833d5ad8b33)

---

## 多线程处理

目前的服务器是单线程版本，只能依次处理用户的请求。如果有一个请求耗时较长，那么在其之后的请求会被阻塞，直到当前请求处理完成才会开始处理下一个请求

#### 模拟缓慢请求

```rust
"GET /sleep HTTP/1.1" => {
    thread::sleep(Duration::from_secs(5));

    let err_html_string = fs::read_to_string("err.html").unwrap();
    create_response("HTTP/1.1 404 NOT FOUND", &err_html_string)
}
```

在 `handle_response` 函数内加一条匹配条件，当路径为 /sleep 时延迟 5 秒再返回数据，用于模仿请求速度缓慢的情况

延迟使用 `thread::sleep` 实现，`Duration::from_secs(5)` 创建一个 `Duration` 类型的对象，表示 5 秒的时间

#### 线程池

线程池（Thread Pool）的主要作用是**管理和复用固定数量的线程**，以高效地执行多个并发任务，避免频繁创建和销毁线程所带来的性能开销。在本例中可以使用线程池来对请求进行多线程处理

```rust
pub struct ThreadPool {
    workers: Vec<Worker>,
}

impl ThreadPool {
    pub fn new(thread_count: usize) -> ThreadPool {
        let mut workers = Vec::with_capacity(thread_count);

        for index in 0..thread_count {
            workers.push(Worker::new(
                index.try_into().unwrap(),
            ));
        }

        ThreadPool { workers, sender }
    }
}

struct Worker {
    id: i32,
}

impl Worker {
    fn new(id: i32) -> Worker {
        thread::spawn(|| {});

        Worker { id }
    }
}
```

`ThreadPool` 是一个公共结构体，可以被 `main.rs` 引用

`Worker` 结构体用于创建和存储线程

`ThreadPool::new` 函数创建了一个线程池，根据传入的参数 `thread_count` 确定线程池中线程的数量。首先使用 `Vec::with_capacity` 创建一个向量并预留空间，然后遍历创建若干个 `worker` 存入向量

#### 线程池接口

线程池应提供一个函数，外部调用可以为线程池添加任务，随后线程池内部自动分配具体执行该任务的线程。​在本例中我们创建 `execute` 函数来实现这一功能


```rust
impl ThreadPool {  
    pub fn execute<T>(&self, f: T)
    where
        T: FnOnce() + Send + 'static,
    {}
}
```

`execute` 方法接受一个泛型参数 T，可以是任意符合条件的闭包或函数。T 必须实现 `FnOnce()`（可以被调用一次）、`Send`（可以在线程间安全传递）、`'static`（生命周期不受限）这三个约束条件

#### 创建消息通道

传入线程池的闭包需要在内部进行分配并处理，可以使用消息通道来实现。各线程不断循环使用 `receiver` 接收任务，外部调用 `execute` 函数时使用 `sender` 发送任务

```rust
impl ThreadPool {
      pub fn new(thread_count: usize) -> ThreadPool {
        ...
        let (sender, receiver) = mpsc::channel();
        ...
    }

    pub fn execute<T>(&self, f: T)
    where
        T: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}
```

闭包通过 `sender` 发送到一个通道 (`mpsc::channel`) 中。这个通道内部维护着一个任务队列。通道中的任务类型必须一致，因此我们需要使用 `Box` 对闭包进行包装

将包装后的 `job` 使用 `sender` 发送

#### 为任务分配线程

为任务分配线程用到了消息通道，消息通道中的接收端 `receiver` 同时提供给多个线程使用，这个特性需要使用智能指针实现

```rust
type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    pub fn new(thread_count: usize) -> ThreadPool {
        ...
        let receiver = Arc::new(Mutex::new(receiver));

        for index in 0..thread_count {
            workers.push(Worker::new(
                index.try_into().unwrap(),
                Arc::clone(&receiver),
            ));
        }
        ...
    }
}

impl Worker {
    fn new(id: i32, receiver: Arc<Mutex<Receiver<Job>>>) -> Worker {
        thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();

            job();
        });

        Worker { id }
    }
}
```

`Job` 是一个类型别名，表示由 `box` 包装的任务闭包

`receiver` 使用 `Arc` 和 `Mutex` 包装。`Arc` 可以使 `receiver` 在多个线程中共享，`Mutex` 用于确保在多线程环境下，同一时间只有一个线程可以访问 `receiver`

`index.try_into` 的作用是将 `index` 转为 `usize` 类型

`worker` 创建的每个新线程都进行无限循环，每次循环执行到 `recv()` 会阻塞线程。当接收到任务时，`recv() `返回 `job` 并在后续代码中调用，这样就实现了任务的多线程分配

#### 接入多线程

完成线程池模块的代码后，main.rs 还需要接入这部分代码

```rust
use web_server_7878::ThreadPool;

fn main() {
    ...
    let thread_pool = ThreadPool::new(2);

    for stream in listener.incoming() {
        ...
        thread_pool.execute(|| {
            let request_vec = handle_stream(&stream);
            handle_response(request_vec, stream);
        });
    }
}
```

先使用 `use` 关键字引入线程池结构体，然后创建结构体并调用 `execute` 函数。将旧逻辑中的响应处理函数放入 `execute` 的闭包中执行即可

#### 结果验证

浏览器先访问`127.0.0.1:7878/sleep`，然后立即访问 `127.0.0.1:7878`。由于 `/sleep` 路由设置了 5 秒延迟，接入线程池前 hello 页面的访问会被阻塞 5 秒，接入线程池后则无需等待 `/sleep` 处理完成，可以立即访问网站

---

[image](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/be3481b5-ef6a-48d2-bb68-9530dce6a2a2)

---

[image](https://app.capacities.io/fcb41f3d-eb26-439e-860c-4d11e4be6bfb/6a25730e-7e52-4c9d-9974-83f01b9b7b84)

---

## 遗留问题

#### 错误处理

在 Rust 中，unwrap() 常用于快速处理 Result 或 Option 类型的返回值，如果返回了 Err 或 None 就直接引发恐慌（panic）。在生产环境中，通常我们会希望用更优雅的方式来处理错误，特别是涉及多线程操作时，panic 会直接导致线程停止，可能影响程序的稳定性

#### 关闭时的资源清理

如果直接使用 ctrl+c 关闭程序，正在请求服务器的用户将会立即失败。因此在关闭时需要进行一些特殊处理以便正确的释放资源，比如调用 join() 来等待每个工作线程完成等