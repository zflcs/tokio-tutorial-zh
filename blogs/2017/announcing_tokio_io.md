# Announcing the tokio-io Crate

今天，我们很高兴地宣布推出一个新的 crate 和几个可在 Tokio stack 中使用的新工具。这代表了对各种细节进行的一系列并行更新的成果，它们恰好在同一时间顺利完成！简而言之，改进如下：

- 从 [tokio-core](https://crates.io/crates/tokio-core) 中提取的新的 [tokio-io](https://crates.io/crates/tokio-io) crate，弃用 [`tokio_core::io`](https://docs.rs/tokio-core/0.1.9/tokio_core/io/) 模块。
- 将 [bytes](https://crates.io/crates/bytes) crate 引入 [tokio-io](https://crates.io/crates/tokio-io)，允许对 buffering 进行抽象并利用 vectored I/O 等底层功能。
- 向 `Sink` trait 添加一种新方法 `close`，以表达正常关闭。

这些变化改善了 Tokio 的组织和抽象，解决了几个长期存在的问题，并为所有未来的发展提供了稳定的基础。同时，这些更改并不具有破坏性，因为旧的 `io` 模块仍然以弃用形式提供。您可以通过 `cargo update` 立即开始使用所有这些 crate，并使用最新的 `0.1.*` 版本的 crate！

让我们更深入地了解每个变化的详细信息，看看现在有哪些变化。

## Adding a `tokio-io` crate

现有的 [`tokio_core::io`](https://docs.rs/tokio-core/0.1.9/tokio_core/io/) 模块提供了许多有用的抽象，但它们并不是特定于 [tokio-core](https://crates.io/crates/tokio-core) 本身的，而 [tokio-io](https://crates.io/crates/tokio-io) crate 的主要目的是在不涉及运行时的情况下提供这些核心实用工具。使用 [tokio-io](https://crates.io/crates/tokio-io) crate 可以依赖异步 I/O 语义，而无需将自己绑定到特定的运行时，例如 [tokio-core](https://crates.io/crates/tokio-core)。[tokio-io](https://crates.io/crates/tokio-io) crate 旨在与 [`std::io`](https://doc.rust-lang.org/std/io/) 标准库模块类似，为异步生态系统提供通用抽象。[tokio-io](https://crates.io/crates/tokio-io) 中提出的概念和 traits 是 Tokio stack 中所有 I/O 的基础。

[tokio-io](https://crates.io/crates/tokio-io) 的主要内容是 [`AsyncRead`](https://docs.rs/tokio-io/0.1.6/tokio_io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio-io/0.1.6/tokio_io/trait.AsyncWrite.html) traits。这两种 traits 是一种“split [`Io`](https://docs.rs/tokio-core/0.1.9/tokio_core/io/trait.Io.html) trait”，被选中来划分实现类似 Tokio 的读/写语义（非阻塞和通知 future 的任务）的类型。然后，这些 traits 与 [bytes](https://crates.io/crates/bytes) crate 集成，以提供一些方便的功能并保留诸如 [`split`](https://docs.rs/tokio-io/0.1.6/tokio_io/trait.AsyncRead.html#method.split) 之类的旧功能。

我们还借此机会将 [tokio-core](https://crates.io/crates/tokio-core) crate 中的 [`Codec`](https://docs.rs/tokio-core/0.1.9/tokio_core/io/trait.Codec.html) trait 刷新为 [`Encoder`](https://docs.rs/tokio-io/0.1.6/tokio_io/codec/trait.Encoder.html) 和 [`Decoder`](https://docs.rs/tokio-io/0.1.6/tokio_io/codec/trait.Decoder.html) trait，它们可以对 [bytes](https://crates.io/crates/bytes) crate 中的类型进行操作（[`EasyBuf`](https://docs.rs/tokio-core/0.1.9/tokio_core/io/struct.EasyBuf.html) 不存在于 [tokio-io](https://crates.io/crates/tokio-io) 中，并且现在已在 [tokio-core](https://crates.io/crates/tokio-core) 中弃用）。这些类型允许您快速地从字节流移动到准备接受框架消息的 [`Sink`](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html) 和 [`Stream`](https://docs.rs/futures/0.1/futures/stream/trait.Stream.html)。一个很好的例子是，使用 [tokio-io](https://crates.io/crates/tokio-io)，我们可以使用新的 [`length_delimited`](https://docs.rs/tokio-io/0.1.6/tokio_io/codec/length_delimited/index.html) 模块与 [tokio-serde-json](https://github.com/carllerche/tokio-serde-json) 结合来立即启动并运行 JSON RPC 服务器，正如我们将在本文后面看到的那样。

总体而言，借助 [tokio-io](https://crates.io/crates/tokio-io)，我们还能够重新审视所设计 API 中的几个小问题。这反过来又使我们能够解决一系列与 [tokio-core](https://crates.io/crates/tokio-core) [相关的问题](https://github.com/tokio-rs/tokio-core/issues/61#issuecomment-277568977)。我们认为 [tokio-io](https://crates.io/crates/tokio-io) 是 Tokio stack 向前发展的一大补充。如果愿意，Crates 可以选择抽象 [tokio-io](https://crates.io/crates/tokio-io)，而无需引入 [tokio-core](https://crates.io/crates/tokio-core) 等运行时。

## Integration with `bytes`

[tokio-core](https://crates.io/crates/tokio-core) 的一个长期缺陷是其 [`EasyBuf`](https://docs.rs/tokio-core/0.1.9/tokio_core/io/struct.EasyBuf.html) 字节缓冲区类型。这种类型基本上就是它的名字（“简单”缓冲区），但不幸的是，在高性能用例中通常不是您想要的。我们一直希望在这里有一个更好的抽象（和更好的具体实现）。

使用  [tokio-io](https://crates.io/crates/tokio-io) 你会发现 [crates.io](https://crates.io/) 上的 [bytes](https://crates.io/crates/bytes) crate 集成得更加紧密，并且同时提供了高性能和“简单”缓冲区所需的抽象。[bytes](https://crates.io/crates/bytes) crate 的主要内容是 [`Buf`](http://carllerche.github.io/bytes/bytes/trait.Buf.html) 和 [`BufMut`](http://carllerche.github.io/bytes/bytes/trait.BufMut.html) trait。这两个 trait 可以抽象任意字节缓冲区（可读和可写），并且现在与所有异步 I/O 对象上的 [`read_buf`](https://docs.rs/tokio-io/0.1.6/tokio_io/trait.AsyncRead.html#method.read_buf) 和 [`write_buf`](https://docs.rs/tokio-io/0.1.6/tokio_io/trait.AsyncWrite.html#method.write_buf) 集成。

除了对多种缓冲区进行抽象的 trait 之外，[bytes](https://crates.io/crates/bytes) crate 还附带了这些 trait 的两个高质量实现，即 [`Bytes`](http://carllerche.github.io/bytes/bytes/struct.Bytes.html) 和 [`BytesMut`](http://carllerche.github.io/bytes/bytes/struct.BytesMut.html) 类型（分别实现 [`Buf`](http://carllerche.github.io/bytes/bytes/trait.Buf.html) 和 [`BufMut`](http://carllerche.github.io/bytes/bytes/trait.BufMut.html) 特征）。简而言之，这些类型代表引用计数缓冲区，允许以有效的方式零拷贝提取数据切片。此外，它们还支持各种常见操作，例如小型缓冲区（内联存储）、单一所有者（可以在内部使用 `Vec`）、具有不相交视图的共享所有者（`BytesMut`）以及具有可能重叠视图的共享所有者（`Bytes`）。

总的来说，我们希望 [bytes](https://crates.io/crates/bytes) crate 是您的一站式字节缓冲区抽象以及高质量实现，让您快速运行。我们很高兴看到[bytes](https://crates.io/crates/bytes) crate 有什么！

## Addition of `Sink::close`

我们最近实现的最后一个重大变化是在 [`Sink`](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html) trait 上添加一种新方法，[`close`](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html#method.close)。到目前为止，还没有关于以通用方式实现 "graceful shutdown" 的精彩故事，因为没有清晰的方法向接收器指示不再有项目被推入。新的 [`close`](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html#method.close) 方法正是为此目的而设计的。

[`close`](https://docs.rs/futures/0.1/futures/sink/trait.Sink.html#method.close) 方法允许通知接收器不再有消息向其推送。接收器可以利用此机会刷新消息，否则将执行特定于协议的关闭。例如，此时 TLS 连接将启动关闭操作，或者代理连接可能会发出 TCP 级关闭。通常，这最终会触底转向新的 [`AsyncWrite::shutdown`](https://docs.rs/tokio-io/0.1.6/tokio_io/trait.AsyncWrite.html#tymethod.shutdown) 方法。

## Addition of `codec::length_delimited`

[tokio-io](https://crates.io/crates/tokio-io) 的一个重要功能是增加了 [`length_delimited`](https://docs.rs/tokio-io/0.1.6/tokio_io/codec/length_delimited/index.html) 模块（受 Netty 的 [`LengthFieldBasedFrameDecoder`](https://netty.io/4.0/api/io/netty/handler/codec/LengthFieldBasedFrameDecoder.html) 启发）。许多协议使用包含帧长度的帧头来分隔帧。举一个简单的例子，采用使用 u32 帧头来分隔帧有效负载的协议。线路上的每个帧如下所示：

```
+----------+--------------------------------+
| len: u32 |          frame payload         |
+----------+--------------------------------+
```

使用 [`length_delimited::Framed`](https://docs.rs/tokio-io/0.1.6/tokio_io/codec/length_delimited/struct.Framed.html) 可以轻松解析此协议：

```rust
// Bind a server socket
let socket = TcpStream::connect(
    &"127.0.0.1:17653".parse().unwrap(),
    &handle);

socket.and_then(|socket| {
    // Delimit frames using a length header
    let transport = length_delimited::FramedWrite::new(socket);
})
```

在上面的例子中，`transport` 将是缓冲区值的 `Sink + Stream`，其中每个缓冲区包含帧有效负载。这使得使用 [serde](https://serde.rs/) 之类的工具将帧编码和解码为一个值变得相当容易。例如，使用 [tokio-serde-json](https://github.com/carllerche/tokio-serde-json)，我们可以快速实现基于 JSON 的协议，其中每个帧的长度都是分隔的，并且帧有效负载使用 JSON 进行编码：

```rust
// Bind a server socket
let socket = TcpStream::connect(
    &"127.0.0.1:17653".parse().unwrap(),
    &handle);

socket.and_then(|socket| {
    // Delimit frames using a length header
    let transport = length_delimited::FramedWrite::new(socket);

    // Serialize frames with JSON
    let serialized = WriteJson::new(transport);

    // Send the value
    serialized.send(json!({
        "name": "John Doe",
        "age": 43,
        "phones": [
            "+44 1234567",
            "+44 2345678"
        ]
    }))
})
```

完整示例在[此处](https://github.com/carllerche/tokio-serde-json/tree/master/examples)。

[`length_delimited`](https://docs.rs/tokio-io/0.1.6/tokio_io/codec/length_delimited/index.html) 模块包含足够的配置设置来处理具有更复杂帧头的长度分隔帧的解析，例如 HTTP/2.0 协议。

## What's next?

所有这些变化加在一起解决了 [futures](https://crates.io/crates/futures) 和 [tokio-core](https://crates.io/crates/tokio-core) crate 中的大量问题，并且我们认为 Tokio 的定位恰好符合我们期望的常见 I/O 和缓冲抽象。与往常一样，我们很乐意听取有关问题跟踪器的反馈，如果您发现问题，我们非常愿意合并 PR！否则，我们期待在实践中看到所有这些变化！

随着 [tokio-core](https://crates.io/crates/tokio-core)、[tokio-io](https://crates.io/crates/tokio-io)、[tokio-service](https://crates.io/crates/tokio-service) 和 [tokio-proto](https://crates.io/crates/tokio-proto) 基础的巩固，Tokio 团队期待着适应和实施更雄心勃勃的协议，如 HTTP/2。我们正在与 [@seanmonstar](https://github.com/seanmonstar) 和 [Hyper](https://github.com/hyperium/hyper) 密切合作，开发这些基础 HTTP 库。最后，我们希望在不久的将来扩展与 HTTP 和通用 [tokio-service](https://crates.io/crates/tokio-service) 实现相关的中间件故事。更多内容即将发布！
