# Refactor I/O driver

描述 Tokio 0.3 版本的 I/O 驱动程序的变化。

## Goals

- 支持使用 &self 为参数的 I/O 类型的 async fn。
- 完善 `Registration` API。

## Non-goals

为 `&TcpStream` 或其他参考类型实现 `AsyncRead` / `AsyncWrite`。

## Overview

目前，I/O 类型要求 `async` 函数使用 `&mut self`。原因是任务的 waker 存储在 I/O 资源的内部状态 (`ScheduledIo`) 中，而不是 `async` 函数返回的 future 中。由于这个限制，I/O 类型将 waker 的数量限制为每个方向一个（一个方向要么是读相关事件，要么是写相关事件）。

将 waker 从内部 I/O 资源的状态移至操作的 future，使得每个操作可以注册多个 waker。`Notify` 使用的“侵入式唤醒列表”策略适用于这种情况，尽管存在一些 I/O 驱动程序特有的问题。

## Reworking the `Registration` type

虽然 `Registration` 是私有的（根据#2728），但它仍然在 Tokio 中作为支持 I/O 资源（例如 `TcpStream`）的实现细节。`Registration` 的 API 已更新，支持等待使用 `&self` 设置的任意 interest。这支持具有不同就绪 interest 的并发等待者。

```rust
struct Registration { ... }

// TODO: naming
struct ReadyEvent {
    tick: u32,
    ready: mio::Ready,
}

impl Registration {
    /// `interest` must be a super set of **all** interest sets specified in
    /// the other methods. This is the interest set passed to `mio`.
    pub fn new<T>(io: &T, interest: mio::Ready) -> io::Result<Registration>
        where T: mio::Evented;

    /// Awaits for any readiness event included in `interest`. Returns a
    /// `ReadyEvent` representing the received readiness event.
    async fn readiness(&self, interest: mio::Ready) -> io::Result<ReadyEvent>;

    /// Clears resource level readiness represented by the specified `ReadyEvent`
    async fn clear_readiness(&self, ready_event: ReadyEvent);
```

为 `T: mio::Evented` 和 `interest` 创建新的注册。这将使用 I/O 驱动程序创建 `ScheduledIo` 条目，并使用 `mio` 注册资源。

由于 Tokio 使用**边缘触发**通知，因此 I/O 驱动程序仅在就绪状态发生**变化**时才从操作系统接收就绪信息。I/O 驱动程序必须跟踪每个资源的已知就绪状态。当进程知道系统调用应该返回 `EWOULDBLOCK` 时，这有助于防止系统调用。

调用 `readiness()` 检查当前已知的资源就绪情况是否与 `interest` 重叠。如果重叠，`readiness()` 会立即返回。如果不重叠，则任务将等待，直到 I/O 驱动程序收到就绪事件。

执行 TCP 读取的伪代码如下。

```rust
async fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
    loop {
        // Await readiness
        let event = self.readiness(interest).await?;

        match self.mio_socket.read(buf) {
            Ok(v) => return Ok(v),
            Err(ref e) if e.kind() == WouldBlock => {
                self.clear_readiness(event);
            }
            Err(e) => return Err(e),
        }
    }
}
```

## Reworking the `ScheduledIo` type

`ScheduledIo` 类型已切换为使用侵入式 waker 链表。链表中的每个条目都包含传递给 `readiness()` 的 `interest` 集。

```rust
#[derive(Debug)]
pub(crate) struct ScheduledIo {
    /// Resource's known state packed with other state that must be
    /// atomically updated.
    readiness: AtomicUsize,

    /// Tracks tasks waiting on the resource
    waiters: Mutex<Waiters>,
}

#[derive(Debug)]
struct Waiters {
    // List of intrusive waiters.
    list: LinkedList<Waiter>,

    /// Waiter used by `AsyncRead` implementations.
    reader: Option<Waker>,

    /// Waiter used by `AsyncWrite` implementations.
    writer: Option<Waker>,
}

// This struct is contained by the **future** returned by `readiness()`.
#[derive(Debug)]
struct Waiter {
    /// Intrusive linked-list pointers
    pointers: linked_list::Pointers<Waiter>,

    /// Waker for task waiting on I/O resource
    waiter: Option<Waker>,

    /// Readiness events being waited on. This is
    /// the value passed to `readiness()`
    interest: mio::Ready,

    /// Should not be `Unpin`.
    _p: PhantomPinned,
}
```

当从 `mio` 收到 I/O 事件时，将更新相关资源的就绪状态并迭代 waiter 列表。所有与收到的就绪事件重叠的 `interest` 的 waiter 都会收到通知。任何 `interest` 与就绪事件不重叠的 waiter 都会保留在列表中。

## Cancel interest on drop

`readiness()` 返回的 future 使用侵入式链表来存储具有 `ScheduledIo` 的 waker。因为 `readiness()` 可以并发调用，所以许多 waker 可能会同时存储在列表中。如果 `readiness()` future 提前被丢弃，则必须将 waker 从列表中删除。这可以防止内存泄漏。

## Race condition

考虑有多少个任务可能同时尝试 I/O 操作。这与 Tokio 如何使用边缘触发事件相结合，可能会导致竞争条件。让我们重新审视 TCP 读取函数：

```rust
async fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
    loop {
        // Await readiness
        let event = self.readiness(interest).await?;

        match self.mio_socket.read(buf) {
            Ok(v) => return Ok(v),
            Err(ref e) if e.kind() == WouldBlock => {
                self.clear_readiness(event);
            }
            Err(e) => return Err(e),
        }
    }
}
```

如果不小心，如果在 `mio_socket.read(buf)` 返回和调用 `clear_readiness(event)` 之间，一个就绪事件到达，`read()` 函数可能会死锁。发生这种情况的原因是收到了就绪事件，`clear_readiness()` 取消设置了就绪事件，并且在下一次迭代中，`readiness().await` 将永远阻塞，因为没有收到新的就绪事件。

当前 I/O 驱动程序通过始终在执行操作之前注册任务的 waker 来处理这种情况。这并不理想，因为它会导致不必要的任务通知。

相反，我们将使用一种策略来防止在收到“未见”的就绪事件时清除就绪状态。I/O 驱动程序将维护一个“tick”值。每次调用 `mio` `poll()` 函数时，tick 都会递增。每个就绪事件都有一个关联的 tick。当 I/O 驱动程序设置资源的就绪状态时，驱动程序的 tick 会被打包到原子 `usize` 中。

`ScheduledIo` 就绪状态 `AtomicUsize` 的结构如下：

```
| shutdown | generation |  driver tick | readiness |
|----------+------------+--------------+-----------|
|   1 bit  |   7 bits   +    8 bits    +  16 bits  |
```

`shutdown` 和 `generation` 组件目前都存在。

`readiness()` 函数返回一个 `ReadyEvent` 值。此值包括使用资源的就绪值读取的 `tick` 组件。调用 `clear_readiness()` 时，会提供 `ReadyEvent`。仅当当前 `tick` 与 `ReadyEvent` 中包含的 `tick` 匹配时，才会清除 Readiness。如果 tick 值不匹配，则下次迭代时对 `readiness()` 的调用将不会阻塞，并且新 `tick` 将包含在新的 `ReadyToken` 中。

## Implementing `AsyncRead` / `AsyncWrite`

`AsyncRead` 和 `AsyncWrite` traits 使用基于“poll”的 API。这意味着无法使用侵入式链表来跟踪 waker。此外，该操作没有相关的 future，这意味着无法取消对准备就绪事件的 interest。

为了实现 `AsyncRead` 和 `AsyncWrite`，`ScheduledIo` 为读取方向和写入方向包含了专用的 waker。这些值用于存储 waker。不会跟踪 `AsyncRead` 和 `AsyncWrite` 实现的特定 `interest`。假设仅关注以下事件：

- 读就绪
- 读关闭
- 写就绪
- 写关闭

请注意，“读关闭”和“写关闭”仅适用于 Mio 0.7。使用 Mio 0.6 时，情况有点混乱。

只能为资源类型本身实现 `AsyncRead` 和 `AsyncWrite`，而不能为 `&Resource` 实现。为 `&Resource` 实现 trait 将允许对资源进行并发操作。由于每个方向仅存储一个 waker，因此任何并发使用都会导致死锁。另一种实现方法是调用 `Vec<Waker>`，但这会导致内存泄漏。

## Enabling reads and writes for `&TcpStream`

没有为 `&TcpStream` 实现 `AsyncRead` 和 `AsyncWrite`，而是向 `TcpStream` 添加了一个新的函数。

```rust
impl TcpStream {
    /// Naming TBD
    fn by_ref(&self) -> TcpStreamRef<'_>;
}

struct TcpStreamRef<'a> {
    stream: &'a TcpStream,

    // `Waiter` is the node in the intrusive waiter linked-list
    read_waiter: Waiter,
    write_waiter: Waiter,
}
```

现在，可以在 `TcpStreamRef<'a>` 上实现 `AsyncRead` 和 `AsyncWrite`。当删除 `TcpStreamRef` 时，所有相关的唤醒资源都将被清除。

## Removing all the `split()` functions

有了 `TcpStream::by_ref()`，就不再需要 `TcpStream::split()`。相反，可以执行以下操作。

```rust
let rd = my_stream.by_ref();
let wr = my_stream.by_ref();

select! {
    // use `rd` and `wr` in separate branches.
}
```

也可以将 `TcpStream` 存储在 `Arc` 中。

```rust
let arc_stream = Arc::new(my_tcp_stream);
let n = arc_stream.by_ref().read(buf).await?;
```
