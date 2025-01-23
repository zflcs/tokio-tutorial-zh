# Framing

现在，我们将应用刚刚学到的 I/O 知识并实现 Mini-Redis frame 层。Framing 是获取字节流并将其转换为 frame 的过程。Frame 是在两个对等点之间传输的数据单元。 Redis 协议 frame 定义如下：

```rust
use bytes::Bytes;

enum Frame {
    Simple(String),
    Error(String),
    Integer(u64),
    Bulk(Bytes),
    Null,
    Array(Vec<Frame>),
}
```

请注意 frane 如何仅由数据组成，而没有任何语义。命令解析和执行在更高的层次进行。

对于 HTTP，frame 可能如下所示：

```rust
enum HttpFrame {
    RequestHead {
        method: Method,
        uri: Uri,
        version: Version,
        headers: HeaderMap,
    },
    ResponseHead {
        status: StatusCode,
        version: Version,
        headers: HeaderMap,
    },
    BodyChunk {
        chunk: Bytes,
    },
}
```

为了实现 Mini-Redis 的 framing，我们将实现一个 `Connection` 结构，它包装一个 `TcpStream` 并读取/写入 `mini_redis::Frame`。

```rust
use tokio::net::TcpStream;
use mini_redis::{Frame, Result};

struct Connection {
    stream: TcpStream,
    // ... other fields here
}

impl Connection {
    /// Read a frame from the connection.
    /// 
    /// Returns `None` if EOF is reached
    pub async fn read_frame(&mut self)
        -> Result<Option<Frame>>
    {
        // implementation here
    }

    /// Write a frame to the connection.
    pub async fn write_frame(&mut self, frame: &Frame)
        -> Result<()>
    {
        // implementation here
    }
}
```

您可以在[这里](https://redis.io/topics/protocol)找到 Redis 有线协议的详细信息。完整的 `Connection` 代码可在[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs)找到。

## 读取缓冲区

`read_frame` 方法在返回之前等待接收到整个帧。对 `TcpStream::read()` 单次调用可能会返回任意数量的数据。它可以包含整个帧、部分帧或多个帧。如果接收到部分帧，则会缓冲数据并从套接字读取更多数据。如果接收到多个帧，则返回第一帧，并缓冲其余数据，直到下一次调用 `read_frame` 为止。

如果您还没有创建一个名为 `connection.rs` 的新文件，请创建一个新文件。

```shell
touch src/connection.rs
```

为了实现这一点， `Connection` 需要一个读缓冲区字段。数据从套接字读入读缓冲区。当解析一帧时，相应的数据就会从缓冲区中删除。

我们将使用 [`BytesMut`](https://docs.rs/bytes/1/bytes/struct.BytesMut.html) 作为缓冲区类型。这是一个可变版本 [`Bytes`](https://docs.rs/bytes/1/bytes/struct.Bytes.html)。

```rust
use bytes::BytesMut;
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

接下来，我们实现 `read_frame()` 方法。

```rust
use tokio::io::AsyncReadExt;
use bytes::Buf;
use mini_redis::Result;

pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        // Attempt to parse a frame from the buffered data. If
        // enough data has been buffered, the frame is
        // returned.
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // There is not enough buffered data to read a frame.
        // Attempt to read more data from the socket.
        //
        // On success, the number of bytes is returned. `0`
        // indicates "end of stream".
        if 0 == self.stream.read_buf(&mut self.buffer).await? {
            // The remote closed the connection. For this to be
            // a clean shutdown, there should be no data in the
            // read buffer. If there is, this means that the
            // peer closed the socket while sending a frame.
            if self.buffer.is_empty() {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        }
    }
}
```

让我们来分解一下。 `read_frame` 方法在循环中运行。首先调用 `self.parse_frame()`。这将尝试从 `self.buffer` 解析 redis frame。如果有足够的数据来解析 frame，则该 frame 将返回给 `read_frame()` 的调用者。否则，我们尝试从套接字读取更多数据到缓冲区中。读取更多数据后，再次调用 `parse_frame()`。这次，如果接收到足够的数据，解析可能会成功。

从流中读取时，返回值 `0` 表示不会再从对等方接收到更多数据。如果读取缓冲区中仍有数据，则表明已接收到部分帧并且连接突然终止。这是一个错误情况并返回 `Err`。

### `Buf` trait

从流中读取时，会调用 `read_buf`。这个读取函数的参数为 [`bytes`](https://docs.rs/bytes/) crate 中实现的 [`BufMut`](https://docs.rs/bytes/1/bytes/buf/trait.BufMut.html)。

首先，考虑如何使用 `read()` 实现相同的读取循环。 可以使用 `Vec<u8>` 代替 `BytesMut`。

```rust
use tokio::net::TcpStream;

pub struct Connection {
    stream: TcpStream,
    buffer: Vec<u8>,
    cursor: usize,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream,
            // Allocate the buffer with 4kb of capacity.
            buffer: vec![0; 4096],
            cursor: 0,
        }
    }
}
```

以及 `Connection` 上的 `read_frame()` 函数：

```rust
use mini_redis::{Frame, Result};

pub async fn read_frame(&mut self)
    -> Result<Option<Frame>>
{
    loop {
        if let Some(frame) = self.parse_frame()? {
            return Ok(Some(frame));
        }

        // Ensure the buffer has capacity
        if self.buffer.len() == self.cursor {
            // Grow the buffer
            self.buffer.resize(self.cursor * 2, 0);
        }

        // Read into the buffer, tracking the number
        // of bytes read
        let n = self.stream.read(
            &mut self.buffer[self.cursor..]).await?;

        if 0 == n {
            if self.cursor == 0 {
                return Ok(None);
            } else {
                return Err("connection reset by peer".into());
            }
        } else {
            // Update our cursor
            self.cursor += n;
        }
    }
}
```

当使用字节数组和 `read` 时，我们还必须维护一个游标来跟踪已缓冲的数据量。我们必须确保将缓冲区的空部分传递给 `read()`。否则，我们将覆盖缓冲的数据。如果缓冲区已满，我们必须增大缓冲区才能继续读取。在 `parse_frame()`（不包括在内）中，我们需要解析 `self.buffer[..self.cursor]` 包含的数据。

由于将字节数组与游标配对非常常见，因此 `bytes` crate 提供了表示字节数组和游标的抽象。`Buf` trait 是由可以读取数据的类型实现的。`BufMut` trait 是由可以写入数据的类型实现的。当传递一个` T: BufMut` 到 `read_buf()`，缓冲区的内部游标会由 `read_buf` 自动更新。因此，在我们的 `read_frame` 版本中，我们不需要管理自己的游标。

此外，当使用 `Vec<u8>` 时，必须初始化缓冲区。 `vec![0; 4096]` 分配一个 4096 字节的数组并向每个条目写入零。调整缓冲区大小时，新容量也必须用零初始化。初始化过程不是免费的。使用 `BytesMut` 和 `BufMut` 时，capacity 未初始化。 `BytesMut` 抽象阻止我们读取未初始化的内存。这让我们可以避免初始化步骤。

## 解析

现在，让我们看一下 `parse_frame()` 函数。解析分两步完成。

1. 确保缓冲完整帧并找到帧的结束索引。
2. 解析帧。

`mini-redis` crate 为我们提供了执行这两个步骤的函数：

1. [`Frame::check`](https://docs.rs/mini-redis/0.4/mini_redis/frame/enum.Frame.html#method.check)
2. [`Frame::parse`](https://docs.rs/mini-redis/0.4/mini_redis/frame/enum.Frame.html#method.parse)

我们还将重用 `Buf` 抽象来提供帮助。一个 `Buf` 被传入 `Frame::check`。当 `check` 函数迭代传入的缓冲区时，内部游标将前进。当 `check` 返回时，缓冲区的内部游标指向帧的末尾。

对于 `Buf` 类型，我们将使用 [`std::io::Cursor<&[u8]>`](https://doc.rust-lang.org/stable/std/io/struct.Cursor.html)。

```rust
use mini_redis::{Frame, Result};
use mini_redis::frame::Error::Incomplete;
use bytes::Buf;
use std::io::Cursor;

fn parse_frame(&mut self)
    -> Result<Option<Frame>>
{
    // Create the `T: Buf` type.
    let mut buf = Cursor::new(&self.buffer[..]);

    // Check whether a full frame is available
    match Frame::check(&mut buf) {
        Ok(_) => {
            // Get the byte length of the frame
            let len = buf.position() as usize;

            // Reset the internal cursor for the
            // call to `parse`.
            buf.set_position(0);

            // Parse the frame
            let frame = Frame::parse(&mut buf)?;

            // Discard the frame from the buffer
            self.buffer.advance(len);

            // Return the frame to the caller.
            Ok(Some(frame))
        }
        // Not enough data has been buffered
        Err(Incomplete) => Ok(None),
        // An error was encountered
        Err(e) => Err(e.into()),
    }
}
```

完整的 [`Frame::check`](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/frame.rs#L65-L103) 函数可以在[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/frame.rs#L65-L103)找到。我们不会完整地介绍它。

需要注意的相关事项是使用了 `Buf` 的“字节迭代器”风格 API。它们获取数据并推进内部游标。例如，为了解析帧，检查第一个字节以确定帧的类型。使用的函数是 `Buf::get_u8`。这将获取当前游标位置处的字节并将游标前进一位。

关于 [`Buf`](https://docs.rs/bytes/1/bytes/buf/trait.Buf.html) trait 还有更多有用的方法。检查 [`API 文档`](https://docs.rs/bytes/1/bytes/buf/trait.Buf.html) 了解更多详情。

## 写入缓冲

Framing API 的另一半是 `write_frame(frame)` 函数。该函数将整个帧写入套接字。为了尽量减少 `write` 系统调用，写入将被缓冲。维护了一个写的缓冲区，在写入套接字之前编码到此缓冲区。然而，与 `read_frame()` 不同，在写入套接字之前，整个帧并不总是缓冲到字节数组中。

考虑一个批处理帧。写入的值是 `Frame::Bulk(Bytes)`。批处理帧的格式是由 $ 字符开始，后跟数据长度（以字节为单位）。帧的大部分内容是 `Bytes` 值的内容。如果数据很大，将其复制到中间缓冲区的成本将会很高。

为了实现缓冲写入，我们将使用 [`BufWriter`](https://docs.rs/tokio/1/tokio/io/struct.BufWriter.html) 结构。该结构体用 `T: AsyncWrite` 初始化并实现 `AsyncWrite` 本身。当在 `BufWriter` 上调用 `write` 时，写入不会直接写入内部的 writer，而是写入缓冲区。当缓冲区满时，内容将刷新到内部的 writer，并清除内部缓冲区。还有一些优化允许在某些情况下绕过缓冲区。

作为本教程的一部分，我们不会尝试完整实现 `write_frame()`。请参阅[此处](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L159-L184)的完整实施。

首先，更新 `Connection` 结构：

```rust
use tokio::io::BufWriter;
use tokio::net::TcpStream;
use bytes::BytesMut;

pub struct Connection {
    stream: BufWriter<TcpStream>,
    buffer: BytesMut,
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream: BufWriter::new(stream),
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

接下来，实现 `write_frame()`。

```rust
use tokio::io::{self, AsyncWriteExt};
use mini_redis::Frame;

async fn write_frame(&mut self, frame: &Frame)
    -> io::Result<()>
{
    match frame {
        Frame::Simple(val) => {
            self.stream.write_u8(b'+').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Error(val) => {
            self.stream.write_u8(b'-').await?;
            self.stream.write_all(val.as_bytes()).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Integer(val) => {
            self.stream.write_u8(b':').await?;
            self.write_decimal(*val).await?;
        }
        Frame::Null => {
            self.stream.write_all(b"$-1\r\n").await?;
        }
        Frame::Bulk(val) => {
            let len = val.len();

            self.stream.write_u8(b'$').await?;
            self.write_decimal(len as u64).await?;
            self.stream.write_all(val).await?;
            self.stream.write_all(b"\r\n").await?;
        }
        Frame::Array(_val) => unimplemented!(),
    }

    self.stream.flush().await;

    Ok(())
}
```

这里使用的函数由 [`AsyncWriteExt`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html) 提供。`TcpStream` 也可以使用这些方法，但不建议在没有中间缓冲区的情况下发出单字节写入。

- [`write_u8`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_u8) 将单个字节写入 writer。
- [`write_all`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWriteExt.html#method.write_all) 将整个切片写入 writer。
- [`write_decimal`](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L225-L238) 是由 mini-redis 实现的。


该函数以调用 `self.stream.flush().await` 结束。因为 `BufWriter` 将写入存储在中间缓冲区中，调用 `write` 不能保证数据写入套接字。在返回之前，我们希望将帧写入套接字。对 `flush()` 调用将缓冲区中挂起的任何数据写入套接字。

另一种选择是不在 `write_frame()` 中调用 `flush()`。相反，在 `Connection` 上提供 `flush()` 函数。这将允许调用者在写入缓冲区中写入多个小帧，然后通过一个 `write` 系统调用将它们全部写入套接字。这样做会使 `Connection` API 变复杂。简单是 Mini-Redis 的目标之一，因此我们决定在 `fn write_frame()` 中包含 `flush().await` 调用。
