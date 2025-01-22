# I/O

Tokio 中的 I/O 操作方式与 `std` 中的操作方式非常相似，但是是异步的。有一个用于读取的 trait ( [`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html) ) 和一个用于写入的 trait ( [`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html) )。这些类型（ [`TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html) 、 [`File`](https://docs.rs/tokio/1/tokio/fs/struct.File.html) 、 [`Stdout`](https://docs.rs/tokio/1/tokio/io/struct.Stdout.html) ）实现了这些 Trait。许多数据结构也实现了 [`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html)，例如 `Vec<u8>` 和 `&[u8]`。这允许在需要 reader 或 writer 的地方使用 byte arrays。

本页将介绍使用 Tokio 进行基本的 I/O 读取和写入，并通过一些示例进行操作。下一页将介绍更高级的 I/O 示例。

## `AsyncRead` 和 `AsyncWrite`

这两个 Trait 提供了异步读取和写入字节流的设施。这些 trait 的方法通常不会直接调用，类似于您不手动调用 `Future` trait 的 `poll` 方法。相反，您将通过 [`AsyncReadExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html) 和 [`AsyncWriteExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html) 提供的方法来间接调用。

让我们简要地看一下其中的一些方法。所有这些功能都是 `async` 的并且必须与 `.await` 一起使用。

### `async fn read()`

[`AsyncReadExt::read`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read) 提供了一个异步方法，用于将数据读入缓冲区，并返回读取的字节数。

注意：当 `read()` 返回 `Ok(0)` 时，表示流已关闭。对 `read()` 任何进一步调用都将立即通过 `Ok(0)` 完成。对于 [`TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html) 实例，这表示套接字的读取部分已关闭。

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = [0; 10];

    // read up to 10 bytes
    let n = f.read(&mut buffer[..]).await?;

    println!("The bytes: {:?}", &buffer[..n]);
    Ok(())
}
```

### `async fn read_to_end()`

[`AsyncReadExt::read_to_end`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read_to_end) 从流中读取所有字节，直到 EOF。

```rust
use tokio::io::{self, AsyncReadExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut f = File::open("foo.txt").await?;
    let mut buffer = Vec::new();

    // read the whole file
    f.read_to_end(&mut buffer).await?;
    Ok(())
}
```

### `async fn write()`

[`AsyncWriteExt::write`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write) 将缓冲区写入 writer，返回写入的字节数。

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    // Writes some prefix of the byte string, but not necessarily all of it.
    let n = file.write(b"some bytes").await?;

    println!("Wrote the first {} bytes of 'some bytes'.", n);
    Ok(())
}
```

### `async fn write_all()`

[`AsyncWriteExt::write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all) 将整个缓冲区写入 writer。

```rust
use tokio::io::{self, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut file = File::create("foo.txt").await?;

    file.write_all(b"some bytes").await?;
    Ok(())
}
```

这两个 trait 都包含许多其他有用的方法。请参阅 API 文档以获取完整列表。

## 辅助函数

此外，就像 `std` 一样，[`tokio::io`](https://docs.rs/tokio/1/tokio/io/index.html) 模块包含许多有用的实用函数以及用于处理 [standard input](https://docs.rs/tokio/1/tokio/io/fn.stdin.html)、[standard output](https://docs.rs/tokio/1/tokio/io/fn.stdout.html) 以及 [standard error](https://docs.rs/tokio/1/tokio/io/fn.stderr.html) 的API。例如，[`tokio::io::copy`](https://docs.rs/tokio/1/tokio/io/fn.copy.html) 将 reader 的全部内容异步复制到 writer 中。

```rust
use tokio::fs::File;
use tokio::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let mut reader: &[u8] = b"hello";
    let mut file = File::create("foo.txt").await?;

    io::copy(&mut reader, &mut file).await?;
    Ok(())
}
```

请注意，这使用了字节数组也实现 `AsyncRead`。

## Echo server

让我们练习一些异步 I/O。我们将编写一个 Echo server。

echo 服务器绑定一个 `TcpListener` 并循环接收入站连接。对于每个入站连接，都会从套接字读取数据并立即写回到套接字。客户端向服务器发送数据并接收回完全相同的数据。

我们将使用略有不同的策略来实现两次 echo 服务器。

### 使用 `io::copy()`

首先，我们将使用 [`io::copy`](https://docs.rs/tokio/1/tokio/io/fn.copy.html) 工具实现回显逻辑。

您可以在新的二进制文件中编写此代码：

```shell
touch src/bin/echo-server-copy.rs
```

您可以使用以下命令启动（或仅检查编译）：

```shell
cargo run --bin echo-server-copy
```

您将能够使用标准命令行工具（例如 `telnet`）或编写一个简单的客户端（例如 [`tokio::net::TcpStream`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#examples) 文档中找到的客户端）来尝试使用服务器。

这是一个 TCP 服务器，需要一个 accept 循环。生成一个新任务来处理每个接收的套接字。

```rust
use tokio::io;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            // Copy data here
        });
    }
}
```

如前所述，该函数需要一个 reader 和一个 writer，并将数据从 reader 复制到 writer。但是，我们只有一个 `TcpStream`，但它同时实现了 `AsyncRead` 和 `AsyncWrite`。因为 `io::copy` 要求 reader 和 writer 都使用 `&mut`，套接字不能同时用于两个参数。

```rust
// This fails to compile
io::copy(&mut socket, &mut socket).await
```

### 拆分 reader + writer

为了解决这个问题，我们必须将套接字拆分为 reader 句柄和 writer 句柄。拆分 reader/writer 组合的最佳方法取决于具体类型。

任何 reader + writer 类型都可以使用 [`io::split`](https://docs.rs/tokio/1/tokio/io/fn.split.html) 进行拆分。该函数接收单个参数并返回单独的 reader 和 writer 句柄。这两个句柄可以独立使用，包括在单独的任务中使用。

例如，echo 客户端可以像这样处理并发读取和写入：

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> io::Result<()> {
    let socket = TcpStream::connect("127.0.0.1:6142").await?;
    let (mut rd, mut wr) = io::split(socket);

    // Write data in the background
    tokio::spawn(async move {
        wr.write_all(b"hello\r\n").await?;
        wr.write_all(b"world\r\n").await?;

        // Sometimes, the rust type inferencer needs
        // a little help
        Ok::<_, io::Error>(())
    });

    let mut buf = vec![0; 128];

    loop {
        let n = rd.read(&mut buf).await?;

        if n == 0 {
            break;
        }

        println!("GOT {:?}", &buf[..n]);
    }

    Ok(())
}
```

由于 `io::split` 支持任何实现 `AsyncRead + AsyncWrite` 并返回独立句柄的值，因此需要在 `io::split` 内部使用 `Arc` 和 `Mutex`。使用 `TcpStream` 可以避免这种开销。`TcpStream` 提供两种专门的分割功能。

[`TcpStream::split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.split) 获取对流的引用并返回 reader 和 writer 句柄。由于使用了引用，因此两个句柄必须保留在调用 `split()` 的相同任务内。这种专门的 `split` 是零成本的。不需要 `Arc` 或 `Mutex`。`TcpStream` 还提供 [`into_split`](https://docs.rs/tokio/1/tokio/net/struct.TcpStream.html#method.into_split) 它支持可以跨任务移动的句柄，而只需花费一个 `Arc`。

因为 `io::copy()` 是在拥有 `TcpStream` 的同一任务上调用的，所以我们可以使用 `TcpStream::split`。服务器中处理 echo 逻辑的任务变为：

```rust
tokio::spawn(async move {
    let (mut rd, mut wr) = socket.split();
    
    if io::copy(&mut rd, &mut wr).await.is_err() {
        eprintln!("failed to copy");
    }
});
```

您可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server-copy.rs)找到完整的代码。

### 手动复制

现在让我们看看如何通过手动复制数据来编写 echo 服务器。为此，我们使用 [`AsyncReadExt::read`](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html#method.read) 和 [`AsyncWriteExt::write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all) 。

完整的 echo 服务器如下：

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:6142").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    // Return value of `Ok(0)` signifies that the remote has
                    // closed
                    Ok(0) => return,
                    Ok(n) => {
                        // Copy the data back to socket
                        if socket.write_all(&buf[..n]).await.is_err() {
                            // Unexpected socket error. There isn't much we can
                            // do here so just stop processing.
                            return;
                        }
                    }
                    Err(_) => {
                        // Unexpected socket error. There isn't much we can do
                        // here so just stop processing.
                        return;
                    }
                }
            }
        });
    }
}
```

（您可以将此代码放入 `src/bin/echo-server.rs` 并使用以下命令启动它  `cargo run --bin echo-server`）。

让我们来分解一下。首先，由于使用了 `AsyncRead` 和 `AsyncWrite`，因此必须引入扩展 trait。

```rust
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
```

### 分配缓冲区

该策略是将一些数据从套接字读入缓冲区，然后将缓冲区的内容写回套接字。

```rust
let mut buf = vec![0; 1024];
```

显式避免使用堆栈缓冲区。回想一下[前面](https://tokio.rs/tokio/tutorial/spawning#send-bound)的内容，我们注意到，跨 `.await` 调用的所有任务数据都必须由任务存储。在这种情况下，`buf` 在 `.await` 调用中使用。所有任务数据都存储在单个分配中。您可以将其视为一个 `enum`，其中每个变体都是特定调用 `.await` 时需要存储的数据。

如果缓冲区由堆栈数组表示，则每个接收的套接字生成的任务的内部结构可能类似于：

```rust
struct Task {
    // internal task fields here
    task: enum {
        AwaitingRead {
            socket: TcpStream,
            buf: [BufferType],
        },
        AwaitingWriteAll {
            socket: TcpStream,
            buf: [BufferType],
        }

    }
}
```

如果使用堆栈数组作为缓冲区类型，它将内联存储在任务结构中。这会让任务结构变得非常庞大。此外，缓冲区大小通常是页大小。反过来，这将使 `Task` 大小变得尴尬： `$page-size + a-few-bytes`。

编译器比基本 `enum` 进一步优化了异步块的布局。实际上，变量不会像 `enum` 所要求的那样在变体之间移动。但是，任务结构大小至少与最大变量一样大。

因此，为缓冲区使用专用分配通常会更有效。

### 处理 EOF

当 TCP 流的读取部分关闭时，对 `read()` 调用将返回 `Ok(0)`。此时退出读取循环很重要。忘记在 EOF 上中断读取循环是错误的常见来源。

```rust
loop {
    match socket.read(&mut buf).await {
        // Return value of `Ok(0)` signifies that the remote has
        // closed
        Ok(0) => return,
        // ... other cases handled here
    }
}
```

忘记从读取循环中中断通常会导致 100% CPU 无限循环情况。当套接字关闭时， `socket.read()` 立即返回，循环将永远重复。

完整代码可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/io/src/echo-server.rs)找到。
