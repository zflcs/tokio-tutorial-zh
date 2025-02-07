# Unit Testing

本页的目的是提供有关如何在异步应用程序中编写有用的单元测试的建议。

### Pausing and resuming time in tests

有时，异步代码通过调用 [`tokio::time::sleep`](https://docs.rs/tokio/1/tokio/time/fn.sleep.html) 或等待 [`tokio::time::Interval::tick`](https://docs.rs/tokio/1/tokio/time/struct.Interval.html#method.tick) 来明确等待。当单元测试开始运行非常缓慢时，基于时间的测试行为（例如，指数退避）会变得麻烦。然而，在内部，tokio 的时间相关功能支持暂停和恢复时间。暂停时间的效果是任何与时间相关的 Future 都可能提前准备好。与时间相关的 Future 提前解决的条件是没有其他可能准备就绪的 Future。当唯一等待的 Future 与时间相关时，这本质上是快进时间：

```rust
#[tokio::test]
async fn paused_time() {
    tokio::time::pause();
    let start = std::time::Instant::now();
    tokio::time::sleep(Duration::from_millis(500)).await;
    println!("{:?}ms", start.elapsed().as_millis());
}
```

此代码在合理的机器上打印 `0ms`。

有关更多详细信息，请参阅 [tokio::test “配置运行时以暂停时间启动”](https://docs.rs/tokio/latest/tokio/attr.test.html#configure-the-runtime-to-start-with-time-paused)。

当然，即使使用不同的时间相关的 Future，Future 解决的时间顺序也会得到保持：

```rust
#[tokio::test(start_paused = true)]
async fn interval_with_paused_time() {
    let mut interval = interval(Duration::from_millis(300));
    let _ = timeout(Duration::from_secs(1), async move {
        loop {
            interval.tick().await;
            println!("Tick!");
        }
    })
    .await;
}
```

此代码立即打印 `"Tick!"`，恰好 4 次。

### 使用 [`AsyncRead`](https://docs.rs/tokio/latest/tokio/io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio/latest/tokio/io/trait.AsyncWrite.html) 进行模拟

异步读取和写入的通用 traits（[`AsyncRead`](https://docs.rs/tokio/latest/tokio/io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio/latest/tokio/io/trait.AsyncWrite.html)）由套接字等实现。它们可用于模拟套接字执行的 I/O。

考虑一下设置这个简单的 TCP 服务器循环：

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    loop {
        let Ok((mut socket, _)) = listener.accept().await else {
            eprintln!("Failed to accept client");
            continue;
        };

        tokio::spawn(async move {
            let (reader, writer) = socket.split();
            // Run some client connection handler, for example:
            // handle_connection(reader, writer)
                // .await
                // .expect("Failed to handle connection");
        });
    }
}
```

这里，每个 TCP 客户端连接都由其专用的 tokio 任务提供服务。此任务拥有一个 reader 和一个 writer，它们从 [`TcpStream`](https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html) 中[分离](https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html#method.split)出来。

现在考虑实际的客户端处理程序任务，特别是函数签名的 where 子句：

```rust
use tokio::io::{AsyncBufReadExt, AsyncRead, AsyncWrite, AsyncWriteExt, BufReader};

async fn handle_connection<Reader, Writer>(
    reader: Reader,
    mut writer: Writer,
) -> std::io::Result<()>
where
    Reader: AsyncRead + Unpin,
    Writer: AsyncWrite + Unpin,
{
    let mut line = String::new();
    let mut reader = BufReader::new(reader);

    loop {
        if let Ok(bytes_read) = reader.read_line(&mut line).await {
            if bytes_read == 0 {
                break Ok(());
            }
            writer
                .write_all(format!("Thanks for your message.\r\n").as_bytes())
                .await
                .unwrap();
        }
        line.clear();
    }
}
```

本质上，给定的 reader 和 writer（实现 [`AsyncRead`](https://docs.rs/tokio/latest/tokio/io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio/latest/tokio/io/trait.AsyncWrite.html)）按顺序提供服务。对于收到的每一行，处理程序都会回复 `"Thanks for your message."`。

为了对客户端连接处理程序进行单元测试，可以使用 [`tokio_test::io::Builder`](https://docs.rs/tokio-test/latest/tokio_test/io/struct.Builder.html) 作为模拟：

```rust
#[tokio::test]
async fn client_handler_replies_politely() {
    let reader = tokio_test::io::Builder::new()
        .read(b"Hi there\r\n")
        .read(b"How are you doing?\r\n")
        .build();
    let writer = tokio_test::io::Builder::new()
        .write(b"Thanks for your message.\r\n")
        .write(b"Thanks for your message.\r\n")
        .build();
    let _ = handle_connection(reader, writer).await;
}
```
