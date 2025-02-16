# Tokio alpha release with async & await

我们很高兴地宣布发布第一个支持 async & await 的 Tokio alpha [版本](https://crates.io/crates/tokio/0.2.0-alpha.1)。这包括更新所有 Tokio 包以使用 `std::future` 而不是 `futures` 0.1。它还包括添加 API 的 `async fn` 版本。

通过向 `Cargo.toml` 添加获取：

```rust
tokio = "=0.2.0-alpha.1"
```

echo 服务器的编写方式如下：

```rust
#![feature(async_await)]

use tokio::net::TcpListener;
use tokio::prelude::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "127.0.0.1:8080".parse()?;
    let mut listener = TcpListener::bind(&addr)?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // In a loop, read data from the socket and write the data back.
            loop {
                let n = match socket.read(&mut buf).await {
                    // socket closed
                    Ok(n) if n == 0 => return,
                    Ok(n) => n,
                    Err(e) => {
                        println!("failed to read from socket; err = {:?}", e);
                        return;
                    }
                };

                // Write the data back
                if let Err(e) = socket.write_all(&buf[0..n]).await {
                    println!("failed to write to socket; err = {:?}", e);
                    return;
                }
            }
        });
    }
}
```

更多示例位于 [git 存储库](https://github.com/tokio-rs/tokio/tree/master/examples)中。

## The road so far

Tokio 首次发布几乎是在[三年前](https://medium.com/@carllerche/announcing-tokio-df6bb4ddb34)。Tokio 首次发布时，它包含一个最小的单线程调度程序和 TCP API。从那时起，它已经发展到包括一个多线程工作窃取调度程序、计时器 API、TCP、UDP、UDS、文件系统访问、信号处理、管理子进程、高性能异步感知通道等等。这一切都要归功于一支[敬业的维护团队](https://github.com/orgs/tokio-rs/people)和[活跃的社区贡献者](https://github.com/tokio-rs/tokio/graphs/contributors)。没有你们，Tokio 就不会有今天的成就！非常感谢。

Tokio 的目标一直是提供一个符合人体工程学的库，用于为 Rust 编写异步应用程序，同时不影响性能。过去三年，我们致力于构建 Rust I/O 生态系统，并探索编写异步 Rust 的习惯用法和模式。在过去三年中，Rust 社区围绕 Tokio 构建了一个充满活力的生态系统。

一些库包括：

- [HTTP 客户端和服务器](https://github.com/hyperium/)（[支持 HTTP/2.0](https://github.com/hyperium/h2)）。
- web 框架（[warp](https://github.com/seanmonstar/warp/)、[actix](https://github.com/actix/actix-web)）。
- [QUIC](https://github.com/djc/quinn) 实现。
- 用于构建强健的服务器和客户端的[可组合组件](https://github.com/tower-rs/tower)。
- [gRPC 客户端和服务器](https://github.com/tower-rs/tower-grpc/)。

（当然，还有很多）。

用户数量也在不断增长！用户包括：

- [Linkerd](https://linkerd.io/)
- [Microsoft](https://github.com/Azure/iotedge)
- [Amazon](https://github.com/firecracker-microvm/firecracker)
- [Facebook](https://github.com/facebookexperimental/mononoke)
- [Vector](https://vector.dev/)
- [Deno](https://github.com/denoland/deno)
- [Libra](https://github.com/libra/libra)
- [PingCAP](https://pingcap.com/)
- [Parity](https://www.parity.io/)

过去三年，Rust 团队也没闲着，他们孜孜不倦地致力于一个能极大改善使用 Rust 编写异步代码体验的重大功能：[async & await 语法](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)。这种新语法即将在 Rust 稳定版中推出。

迄今为止，Tokio 的所有迭代都是在保持 API 稳定性的同时完成的。自 2016 年 9 月 9 日发布第一个 0.1.0 版本以来，Tokio 从未发布过重大版本。可以理解的是，需要进行的更改列表已经很长了。引入 async & await 语法是进行这些重大更改的最佳时机。

## What we have now

今天发布的 Tokio `0.2.0-alpha.1` 带有 async & await 更新了 Tokio 的所有组件以使用 `std::future`，并且所有对 `futures` 0.1 的使用已被删除。

变化包括：

### Using async fns for asynchronous operations.

在可能的情况下，已使用 `async fn` 来提供异步操作。例如，使用 async `accept` 函数从 `TcpListener` 接受套接字：

```rust
let (mut socket, _) = listener.accept().await?;
```

I/O 操作以 `async` 函数的形式提供：

```rust
let n = socket.read(&mut buf).await?;

let n = socket.write(&buf).await?;
```

这同样适用于大多数 Tokio 0.2 API。

### Starting the Tokio runtime

现在可以使用过程宏来设置 Tokio 应用程序的入口点：

```rust
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

仍然支持手动创建运行时。过程宏只是提供了一些糖。等效应用程序将是：

```rust
fn main() {
    let mut rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("Hello from Tokio!");
    });
}
```

还有一个用于测试的过程宏，它将设置当前线程运行时来运行您的测试：

```rust
#[tokio::test]
async fn my_test() {
    let addr = "127.0.0.1:8080".parse().unwrap();
    let mut listener = TcpListener::bind(&addr).unwrap();
    let addr = listener.local_addr().unwrap();

    // Connect to the listener
    TcpStream::connect(&addr).await.unwrap();
}
```

### Nightly only

由于 async 和 await 尚未在稳定的 Rust 频道上提供，因此此 alpha 版本需要使用 nightly Rust。Tokio 包含一个 [rust-toolchain](https://github.com/tokio-rs/tokio/blob/master/rust-toolchain) 文件，可再次跟踪 nightly Tokio 测试。

## Towards a final release

此 alpha 版本代表了 Tokio 迈向下一次迭代的第一步。用户现在可以尝试使用 Tokio 和 async & await 语法，但一切还未确定。

async & await 语法的推出是发布完整 stack 重大修订的最佳时机。Tokio 的所有组件都需要更多更改，包括生态系统包，例如 [`mio`](https://github.com/tokio-rs/mio)、[`bytes`](https://github.com/tokio-rs/bytes) 和 [`tracing`](https://github.com/tokio-rs/tracing)。重点将放在紧密集成和性能上，即在不牺牲速度的情况下提供符合人体工程学的高级 API。过去三年的经验教训将得到应用。后续博客文章将会更详细地介绍这些变化。

随着进展，alpha 版本将会发布。这些版本将包含重大更改。您可以在 [issue tracker](https://github.com/tokio-rs/tokio/issues?q=is%3Aopen+is%3Aissue+milestone%3Av0.2) 上跟踪进度。

考虑到变更的范围，最终版本需要几个月才能准备好。估计软件发布日期很难，但我们的目标是在年底之前。

在代码、测试和文档方面还有大量工作要做。一如既往，非常感谢您的帮助！查看[issue tracker](https://github.com/tokio-rs/tokio/issues?q=is%3Aopen+is%3Aissue+milestone%3Av0.2)或在 [Gitter](https://gitter.im/tokio-rs/tokio) 上 ping 我们。我们很乐意为您提供指导！
