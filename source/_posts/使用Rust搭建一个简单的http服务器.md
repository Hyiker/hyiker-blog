---
title: 使用Rust编写HTTP服务器
date: 2020-07-07 16:40:42
tags:
 - rust
 - http服务器
keywords: rust http服务器
categories: Rust
description: 花了将近半个月的时间入门rust，现在照着doc.rust-lang.org上的官网教程编写了一个http服务器
summary_img:
---

## 简述

最近通过 rust book 学习 rust，根据最后一章的内容制作了一个简单的异步 http 服务器。

<!-- more -->

### 项目结构

|-- hanabi

|-- .gitignore

|-- 404.html

|-- Cargo.lock

|-- Cargo.toml

|-- hello.html

|-- src

| |-- lib.rs

| |-- main.rs

|-- target

​ |-- ..

### 代码部分

`main.rs`

```rust
fn main() {
    // returns a TcpListener instance(wrapped by Result<T,E>)
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    let pool = ThreadPool::new(4);
    // iterate to fetch the incoming tcp connection
    // store the tcp connection inside stream
    // it's currently synchronized.
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
```

`main`函数部分，我们设计了一个`ThreadPool`来实现同步地接受多个请求，在每次接受到一个`listener.incoming()`的请求时，都把他转化为一个`stream`后在 pool 里进行`execute`，`handle`方法如下：

```rust
fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();
    let (get, sleep) = (b"GET / HTTP/1.1\r\n", b"GET /sleep HTTP/1.1\r\n");
    let (filename, status_line) = if buffer.starts_with(get) {
        ("hello.html", "HTTP/1.1 200 OK\r\n\r\n")
    } else if buffer.starts_with(sleep) {
        thread::sleep(Duration::from_millis(10000));
        ("hello.html", "HTTP/1.1 200 OK\r\n\r\n")
    } else {
        ("404.html", "HTTP/1.1 404 NOT FOUND\r\n\r\n")
    };
    let body = fs::read_to_string(filename).unwrap();
    let response = format!("{}{}", status_line, body);
    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

注：这里需要注意`buffer`开的大小，太小可能导致服务器的一条线程直接死亡（需要处理错误，但是因为直接`unwrap`掉了所以会导致线程挂起）

`lib.rs`

```rust
pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}
```

首先定义前文中需要用到的`ThreadPool`，由负责执行任务的`workers`以及给`worker`派发任务的`sender`组成。

```rust
struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}
```

`Worker`类用于包装`thread`类，以下是`Worker::new()`方法

```rust
fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv().unwrap();
            match message {
                Message::NewJob(job) => {
                    println!("Worker {} got a job; executing", id);
                    job();
                }
                Message::Terminate => {
                    println!("Worker {} was told to stop", id);
                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
```

我们使用`thread::spawn(move || loop)`来产生一个循环的线程，通过不停地获取`receiver`中的请求，通过`match`来对不同的请求作出响应，要注意的是，由于我们的需求是：跨线程共享同一对象、同一时间只能被一条线程引用，因此我们需要使用原子变量`Arc<T>`，来使得变量安全地在线程之间共享，注意`Arc<T>`作为一个类似于指针的作用，是开箱即用的。另外，由于`thread`被`Option<T>`包裹，所以需要使用`Some(thread)`来创建。

```rust
enum Message {
    NewJob(Job),
    Terminate,
}
type Job = Box<dyn FnOnce() + Send + 'static>;
```

这里回忆一下`Box<T>`：类似于`C++`中的**_智能指针_**，`Box`负责从 heap 上分配内存，并且将`T`类型的对象放置于其上（rust 中的对象默认是分配在栈上的）。`dyn`则表示对象是动态分发的基类`trait`；`Send`表示该对象可以在线程间安全地传递 ownership，可以作为**_跨线程共享_**的 marker；`FnOnce`表示该方法只能被调用一次。

接下来我们详解一下`ThreadPool`部分的方法实现：

`ThreadPool::new`

```rust
pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);
        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }
        ThreadPool { workers, sender }
}
```

`ThreadPool`的创建，对于每一个即将进入`workers`队列中的方法我们都会对其进行初始化，这里就不需要每次都去 new 一个 receiver 了，而是可以直接使用`Arc::clone`方法来进行实现。

`ThreadPool::execute`

```rust
pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
```

首先，我们定义了执行方法`f: F`，供我们调用，因为之前`job`方法是分配在堆上的，所以需要使用`Box`包裹起来。

### 小结

跟着官网做完这个例子之后，感觉看书的时候还是有很多知识点其实并不完全了解清楚，所以果然编程还是的多靠实践啊，后续也许会继续进行优化改进，并且同步更新 rust 的学习笔记。
