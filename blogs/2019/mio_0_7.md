# Announcing Mio 0.7-alpha.1

我们很高兴地宣布 [Thomas de Zeeuw](https://github.com/Thomasdezeeuw) 成为 Mio 的新负责人。Mio 是支持 Tokio 和其他 Rust 项目的低级 I/O 可移植性抽象。Thomas 参与了 Mio 0.7 的大部分工作，并将继续领导该 crate 升级到 1.0。他撰写了其余公告。

Mio 0.7 是[多位贡献者](https://github.com/tokio-rs/mio/graphs/contributors?from=2019-03-01&to=2019-12-13)历时约半年的成果。与 Mio 0.6 版相比，0.7 版减少了提供的 API 的大小，以简化实现和使用。API 0.7 版本将接近未来 1.0 版本所提议的 API。该 crate 的范围缩小到提供跨平台事件通知机制和常用类型，如跨线程轮询唤醒和非阻塞网络 I/O 原语。

## Major changes

由于这是一个大型版本，因此这里仅描述一些亮点，所有更改都可以在[更改日志](https://github.com/tokio-rs/mio/blob/master/CHANGELOG.md#070-alpha1)中找到。总体而言，我们进行了大量的 API 更改，以降低实现的复杂性并尽可能地消除开销。

### Wrapping native OS types

在 0.6 版本中，Mio 定义了自己的 `Event` 结构，原生 OS 类型会转换为该结构。这意味着轮询后原生类型（即 `kevent` 或 `epoll_event` 结构）会转换为 Mio 的 `Event` 类型。在 0.7 版本中，`Event` 被改为简单的 `kevent` 或 `epoll_event` 的类型包装器，它提供了方便的方法来获取事件就绪指示器（例如可读或可写）。整个 crate 中都进行了类似的更改。例如，`Poll` 现在只是 Unix 上的文件描述符。

使用 `Events` 结构处理事件的方式也发生了变化。在 0.6.10 版本中，索引访问已被弃用，并被 iteration API 取代。索引访问在 0.7 版本中被完全删除，并且iteration API 被更改为返回对 `Event` 的引用而不是进行复制。`Event` 上的所有方法只需要一个引用，所以这应该不是问题。

### Removal of the user space queue & deprecated types

Mio 0.6 版有一个用户空间队列，可通过 `SetReadiness` 和 `Registration` 类型使用，但它已在 0.7 版中删除，因为它被认为超出了 Mio 的范围。需要用户空间队列的用户可以使用 [crossbeam](https://crates.io/crates/crossbeam) 包中的其中一种队列类型。

用户空间队列的一个用例是从另一个线程唤醒轮询线程（即调用 `Poll::poll`）。为了支持此用例，引入了一种新的 `Waker` 类型。调用 `Waker::wake` 将唤醒关联的 `Poll`。

### Registering & I/O resource usages changes

到目前为止，0.7 版给用户带来的最大变化是 Mio 注册 I/O 源的方式。在 0.6 版中，资源是使用 `Token`、`Ready` 和 `PollOpt` 参数注册的。例如，以下代码将注册具有可读兴趣和边缘触发器的套接字。

```rust
poll.register(&socket, Token(0), Ready::readable(), PollOpt::edge())?;
```

如上所述，`Event` 类型已更改为本机操作系统类型的包装器。反过来，这删除了 `​​Ready` 类型，转而使用 `Event` 上的方法来检查就绪指示器并获取 `Token`。注册中使用的 `Ready` 类型已更改为 `Interest`，以更好地反映其用途。`Interest` API 也进行了更改，以利用（某种程度上）新的关联常量，确保不再可能注册具有空 interests 的事件源。

在 0.7 版中，Mio 将所有源注册为边缘触发器，从而无需 `PollOpt`。边缘触发器的使用说明如下。

在0.6版本中定义如何注册事件源的 trait 称为 `Evented`。它被改为 `Source` 并且现在被称为 `event::Source`，因为该类型位于 `event` 模块内部。`event::Source` 具有与 `Evented` 相同的三种方法：`register`，`reregister` 和 `deregister`，但如上所述进行了更改。

最后，registration 功能从 `Poll` 移至新的 `Registry` 类型，该类型可以通过 `try_cloned` 来注册来自不同线程的源。总之，这意味着 registering 具有可读 interests 的相同套接字现在如下所示：

```rust
poll.registry().register(&socket, Token(0), Interest::READABLE)?;
```

### Moving to edge triggers

如前所述，Mio 现在使用边缘触发器注册所有 I/O 源，这意味着单次触发和电平触发器的用户需要改变他们对事件的响应方式。

问题在于，某些级别和一次性触发器的使用会显示出操作系统之间的差异，而 Mio 无法在没有不可接受的开销的情况下取消这些差异。我们希望 Mio 能够提供良好的跨平台体验，因此我们决定让所有 I/O 源都采用边缘触发。这样，所有平台上的行为都是相同的。

以前使用单次触发器的用户现在应该在接收到事件后取消注册 I/O 源。

电平触发器的用户基本上需要对所有 I/O 操作进行循环。使用边沿触发器时，用户需要在收到事件后执行 I/O。例如，响应读取事件，用户必须从 I/O 源读取直到它阻塞（它返回 `io::ErrorKind::WouldBlock` 类型的 `io::Error`）。仅当操作返回 `WouldBlock` 错误时，Mio 才会报告该 I/O 源上该 `Interest` 的更多事件。以下是如何使用边缘触发器从 `TcpStream` 读取的示例。

```rust
let stream: TcpStream = ...;

// When we polled we received a readable event for our `TcpStream`.

let mut buf = [0; 4096];
// With edge triggers we need to read all data available on the socket.
loop {
    let bytes_read = match stream.read(&mut buf) {
        // Read successful, proceed like normal.
        Ok(bytes_read) => bytes_read,
        // No more data to read at the moment. We will receive another event
        // once more data is available to read.
        Err(ref err) if err.kind() == io::ErrorKind::WouldBlock => break,
        // Got interrupted, try again.
        Err(ref err) if err.kind() == io::ErrorKind::Interrupted => continue,
        // Hit an actual error.
        Err(err) => return Err(err),
    };

    process_byte(&buf[0..bytes_read]);
}
```

**注意**：这需要对已注册事件源上的所有 I/O 操作进行，因此在使用自动化工具转换到新的注册 API 时要小心。

### Addition of Unix socket API

`net` 模块中的新功能是 `UnixListener`、`UnixStream` 和 `UnixDatagram` 类型，它们支持 Unix 套接字，其 API 与标准库中的 API 类似。

这些 API 目前仅支持基于 Unix 的操作系统。

### Removal of deprecated API

在 Mio 0.6 版中，各种类型（例如旧的事件循环相关类型和通道类型）已被弃用。在 0.7 版中，所有弃用的类型均被删除。

### Removed support for OSes

下列操作系统的支持已被放弃：

- Linux 版本低于 2.6.27（和 glibc 2.9），我们使用旧 Linux 版本中不存在的较新 API，特别是 `socket(2)` 和 `epoll_create1(2)` 中的` eventfd(2)``、SOCK_NONBLOCK` 和 `SOCK_CLOEXEC` 选项。
- Fuchsia：没有任何 CI 覆盖，并且我们没有足够的维护人员来对其进行适当的支持。
- Bitrig：该操作系统的开发似乎已经停止，并且 rustc 不再支持它。

### Increased minimum supported Rust version

截至撰写本文时，支持的最低 Rust 版本为 1.36。对于 1.0 版本，我们的目标是 Rust 1.39 版本，因为这是 `async` 变得稳定的版本（这是我们许多依赖的 crate 正在使用的一项主要功能）。因此请保持这些编译器为最新版本！

## Maintainer changes

除了代码之外，我们还进行了一些管理变更。存储库已移至 [Tokio 组织](https://github.com/tokio-rs/mio)，并且有多名新人加入了团队。
