# Hello Tokio

我们将通过写基础的 Tokio 应用来入门，它将会连接 Mini-Redis 服务端，设置 hello 键对应的值为 world。并且使用 Mini-Redis 客户端库来读取这个键。

## 代码

### 生成新的 crate

开始时，先创建一个新的 Rust app：

```shell
cargo new my-redis
cd my-redis
```

### 添加依赖项

接下来，打开 `Cargo.toml` 并添加下列内容至 `[dependencies]`：
```rust
tokio = { version = "1", features = ["full"] }
mini-redis = "0.4"
```

### 编写代码

然后，打开 `main.rs` 并将文件内容替换为：

```rust
use mini_redis::{client, Result};

#[tokio::main]
async fn main() -> Result<()> {
    // Open a connection to the mini-redis address.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Set the key "hello" with value "world"
    client.set("hello", "world".into()).await?;

    // Get key "hello"
    let result = client.get("hello").await?;

    println!("got value from the server; result={:?}", result);

    Ok(())
}
```

确保 Mini-Redis 服务器正在运行。在单独的终端窗口中，运行：

```shell
mini-redis-server
```

如果你尚未安装 mini-redis，你可以使用以下命令进行安装：

```shell
cargo install mini-redis
```

现在，可以运行 `my-redis` 应用：

```shell
cargo run
got value from the server; result=Some(b"world")
```

在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/hello-tokio/src/main.rs)可以找到完整代码。

## 逐行分解

让我们花点时间回顾一下我们刚刚做的事情。代码不多，但发生了很多事情。

```shell
let mut client = client::connect("127.0.0.1:6379").await?;
```

`mini-redis` crate 提供的 `client::connect` 函数，它异步的建立特定的远程地址的 TCP 连接。一旦连接建立，将会返回一个 `client` 的句柄。及时这个操作是异步的执行，但写出来的代码看上去像同步的。唯一提示这个操作是异步的是 `.await` 操作符。

### 什么是异步编程？

大多数计算机程序的执行顺序与编写的顺序相同。执行第一行，然后执行下一行，依此类推。使用同步编程，当程序遇到无法立即完成的操作时，它将阻塞直到该操作完成。例如，建立 TCP 连接需要通过网络与对等方进行交换，这可能需要相当长的时间。在此期间，线程被阻塞。

使用异步编程，不能立即完成的操作将被暂停到后台。线程没有被阻塞，可以继续运行其他任务。一旦操作完成，任务将取消暂停并从中断的地方继续处理。我们之前的例子只有一个任务，所以在暂停期间什么也不会发生，但是异步程序通常有许多这样的任务。

虽然异步编程可以提高应用程序的速度，但通常会导致程序变得更加复杂。异步操作完成后，程序员需要跟踪恢复工作所需的所有状态。从历史上看，这是一项繁琐且容易出错的任务。

### Compile-time green-threading

Rust 使用 [async/await](https://en.wikipedia.org/wiki/Async/await) 的特性实现异步编程。执行异步操作的函数标有 `async` 关键字。在我们的示例中，`connect` 函数定义如下：

```rust
use mini_redis::Result;
use mini_redis::client::Client;
use tokio::net::ToSocketAddrs;

pub async fn connect<T: ToSocketAddrs>(addr: T) -> Result<Client> {
    // ...
}
```

`async fn` 定义了类似于常规同步函数的函数，但是异步地执行。Rust 在编译时将 `async fn` 转换为异步操作的例程。在 `async fn` 中对 `.await` 的任何调用都会将控制权交还给线程。操作在后台处理时，线程可以执行其他工作。

> 虽然其他语言也实现了 [async/await](https://en.wikipedia.org/wiki/Async/await)，但 Rust 采用了独特的方法。首先，Rust 的异步操作是惰性的。这导致运行时语义与其他语言不同。

如果这还不太清楚，别担心。我们将在整个指南中进一步探讨 `async/await`。

### 使用 `async/await`

异步函数的调用方式与任何其他 Rust 函数一样。但是调用这些函数不会导致函数体执行。相反，调用 `async fn` 将返回代表操作的值。从概念上讲，这类似于零参数闭包。要实际运行该操作，您应该对返回值使用 `.await` 运算符。

例如，

```rust
async fn say_world() {
    println!("world");
}

#[tokio::main]
async fn main() {
    // Calling `say_world()` does not execute the body of `say_world()`.
    let op = say_world();

    // This println! comes first
    println!("hello");

    // Calling `.await` on `op` starts executing `say_world`.
    op.await;
}
```

输出：

```shell
hello
world
```

`async fn` 的返回值是实现 [Future](https://doc.rust-lang.org/std/future/trait.Future.html) 特征的匿名类型。

### Async `main` function

用于启动应用程序的主要功能与大多数 Rust 包中的常用功能不同。

1. 它是一个 `async fn`。
2. 它用 #[tokio::main] 注解。

当我们想要进入异步上下文时，使用 `async fn`。但是异步函数必须由[运行时](https://docs.rs/tokio/1/tokio/runtime/index.html)执行。运行时包含异步任务调度程序，提供事件 I/O、计时器等。运行时不会自动启动，因此需要主函数来启动它。

`#[tokio::main]` 是一个宏。它将 `async fn main()` 转换为同步 `fn main()`，后者初始化运行时实例并执行 async main 函数。

例如：

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}
```

被转化为：

```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

Tokio 运行时的详细信息将在稍后介绍。

### Cargo features

本教程依赖启用的完整功能的 Tokio：

```rust
tokio = { version = "1", features = ["full"] }
```

Tokio 具有很多功能（TCP、UDP、Unix 套接字、计时器、同步实用程序、多种调度程序类型等）。并非所有应用程序都需要所有功能。当尝试优化编译时间或最终应用程序占用空间时，应用程序可以决定仅选择它使用的功能。
