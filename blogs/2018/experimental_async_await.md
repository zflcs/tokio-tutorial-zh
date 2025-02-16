# Experimental async / await support for Tokio

如果你还没听说过，`async` / `await` 是 Rust 正在开发的一项重要新功能。它旨在让异步编程变得简单（至少比现在简单一点）。这项工作已经进行了一段时间，今天已经可以在 Rust nightly channel 上使用。

我很高兴地宣布 Tokio 现在有了实验性的 async / await 支持！让我们深入了解一下。

## Getting started

首先，Tokio async/await 支持由一个新的包提供，创意地命名为 [`tokio-async-await`](https://crates.io/crates/tokio-async-await)。这个 crate 是 tokio 上的一个垫片。它包含与 tokio 相同的所有类型和功能（作为重新导出），以及用于 async / await 的其他帮助程序。

要使用 [`tokio-async-await`](https://crates.io/crates/tokio-async-await)，您需要从配置为使用 Rust 2018 版的 crate 中依赖它。它也仅适用于最近的 Rust nightly 版本。

在应用程序的 `Cargo.toml` 中，添加以下内容：

```rust
# At the very top of the file
cargo-features = ["edition"]

# In the `[packages]` section
edition = "2018"

# In the `[dependencies]` section
tokio = {version = "0.1", features = ["async-await-preview"]}
```

然后，在您的应用程序中执行以下操作：

```rust
// The nightly features that are commonly needed with async / await
#![feature(await_macro, async_await, futures_api)]

// This pulls in the `tokio-async-await` crate. While Rust 2018
// doesn't require `extern crate`, we need to pull in the macros.
#[macro_use]
extern crate tokio;

fn main() {
    // And we are async...
    tokio::run_async(async {
        println!("Hello");
    });
}
```

并运行它（使用 nightly）：

```shell
cargo +nightly run
```

您正在使用 Tokio + `async` / `await`！

请注意，要生成 `async` 块，应该使用 `tokio::run_async` 函数（而不是 `tokio::run`）。

## Going deeper

现在，让我们构建一些简单的东西：一个 echo 服务器（耶）。

```rust
// Somewhere towards the top

#[macro_use]
extern crate tokio;

use tokio::net::{TcpListener, TcpStream};
use tokio::prelude::*;

// more to come...

// The main function
fn main() {
  let addr: SocketAddr = "127.0.0.1:8080".parse().unwrap();
  let listener = TcpListener::bind(&addr).unwrap();

    tokio::run_async(async {
        let mut incoming = listener.incoming();

        while let Some(stream) = await!(incoming.next()) {
            let stream = stream.unwrap();
            handle(stream);
        }
    });
}
```

在此示例中，`incoming` 是已接受的 `TcpStream` 值流。我们使用 `async` / `await` 来迭代该流。目前，只有用于等待单个值（future）的语法，因此我们使用 `next` 组合器来获取流中下一个值的 future。这让我们可以使用 `while` 语法迭代流。

一旦我们获得流，它就会被传递给 `handle` 函数进行处理。让我们看看它是如何实现的。

```rust
fn handle(mut stream: TcpStream) {
    tokio::spawn_async(async move {
        let mut buf = [0; 1024];

        loop {
            match await!(stream.read_async(&mut buf)).unwrap() {
                0 => break, // Socket closed
                n => {
                    // Send the data back
                    await!(stream.write_all_async(&buf[0..n])).unwrap();
                }
            }
        }
    });
}
```

就像 `run_async` 一样，有一个 `spawn_async` 函数可以生成异步块作为任务。

然后，为了执行 echo 逻辑，我们从套接字读取到缓冲区并将数据写回到同一套接字。因为我们使用的是 `async` / `await`，所以我们可以使用一个看起来像堆栈分配的数组（它实际上最终位于堆中）。

请注意，`TcpStream` 具有 `read_async` 和 `write_all_async` 函数。这些函数执行的逻辑与 `std` 中 `Read` 和 `Write` traits 上的同步等效函数相同。不同之处在于，它们返回可以等待的 future。

`*_async` 函数在 `tokio-async-await` crate 中通过使用扩展 traits 进行定义。这些 traits 通过 `use tokio::prelude::*;` 行导入。

这只是一个开始，请查看存储库中的[示例](https://github.com/tokio-rs/tokio/blob/master/examples)目录以了解更多信息。甚至有一个使用 [hyper](https://github.com/tokio-rs/tokio/blob/master/tokio-async-await/examples/src/hyper.rs) 的示例。

## Some notes

首先，`tokio-async-await` crate 仅提供对 `async` / `await` 语法的兼容性。它不提供对 `futures` 0.3 crate 的支持。预计用户将继续使用 futures 0.1 以保持与 Tokio 的兼容性。

为了实现这一点，`tokio-async-await` crate 定义了自己的 `await!` 宏。此宏是 `std` 提供的宏之上的垫片，用于等待 `futures` 0.1 futures。这就是兼容层能够保持轻量和无样板的方式。

这只是一个开始。随着时间的推移，`async` / `await` 支持将继续发展和改进，但这足以让每个人都开始行动！
