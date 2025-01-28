# Streams

Stream 是一系列异步的值。它是 Rust 的 [`std::iter::Iterator`](https://doc.rust-lang.org/book/ch13-02-iterators.html) 的异步版本，表现为 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) trait。Streams 可以在 `async` 函数中迭代。它们也可以使用 adapter 进行转化，Tokio 在 [`StreamExt`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html) trait 上提供了许多常见的适配器。

Tokio在单独的 crate 中提供 stream 支持：`tokio-stream`。

```rust
tokio-stream = "0.1"
```

> 目前，Tokio 的 Stream 工具存在于 `tokio-stream` crate 中。一旦在 Rust 标准库中稳定了 Stream trait，Tokio 的 Stream 工具将移至 `tokio` Crate。

## Iteration

目前，Rust 编程语言不支持异步 `for` 循环。使用 `while let` 循环和 [`StreamExt::next()`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.next) 来替代。

```rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let mut stream = tokio_stream::iter(&[1, 2, 3]);

    while let Some(v) = stream.next().await {
        println!("GOT = {:?}", v);
    }
}
```

与迭代器一样，`next()` 方法返回 `Option<T>`，其中 `T` 是 Stream 的值类型。 None 表明 Stream 迭代已终止。

### Mini-Redis broadcast

让我们使用Mini-Redis客户端介绍一个更复杂的示例。

完整代码可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/streams/src/main.rs)找到。

```rust
use tokio_stream::StreamExt;
use mini_redis::client;

async fn publish() -> mini_redis::Result<()> {
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Publish some data
    client.publish("numbers", "1".into()).await?;
    client.publish("numbers", "two".into()).await?;
    client.publish("numbers", "3".into()).await?;
    client.publish("numbers", "four".into()).await?;
    client.publish("numbers", "five".into()).await?;
    client.publish("numbers", "6".into()).await?;
    Ok(())
}

async fn subscribe() -> mini_redis::Result<()> {
    let client = client::connect("127.0.0.1:6379").await?;
    let subscriber = client.subscribe(vec!["numbers".to_string()]).await?;
    let messages = subscriber.into_stream();

    tokio::pin!(messages);

    while let Some(msg) = messages.next().await {
        println!("got = {:?}", msg);
    }

    Ok(())
}

#[tokio::main]
async fn main() -> mini_redis::Result<()> {
    tokio::spawn(async {
        publish().await
    });

    subscribe().await?;

    println!("DONE");

    Ok(())
}
```

生成了一个任务用于将消息发布到 Mini-Redis 服务器的“数字”通道上。然后，在 main 任务上，我们订阅“数字”通道道并显示收到的消息。

订阅后，在返回的订户上调用 [`into_stream()`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Subscriber.html#method.into_stream) 。这将消耗 `Subscriber`，并返回产生到达的消息的 Stream。在开始迭代消息之前，请注意使用 [`tokio::pin!`](https://docs.rs/tokio/1/tokio/macro.pin.html) 将 Stream [pin](https://doc.rust-lang.org/std/pin/index.html) 在堆栈上。在 stream 上调用 `next()` 需要将 Stream pin 住。 `into_stream()` 函数返回未固定的 Stream，我们必须明确 pin 住它才能迭代它。

> 当 Rust 值无法再在内存中移动时，它就被“固定”了。固定值的一个关键属性是可以将指针指向固定数据，并且调用者可以确信指针保持有效。`async/await` 使用此功能来支持跨 `.await` 点借用数据。

如果我们忘记固定 Stream，我们会收到这样的错误：

```rust
error[E0277]: `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>` cannot be unpinned
  --> streams/src/main.rs:29:36
   |
29 |     while let Some(msg) = messages.next().await {
   |                                    ^^^^ within `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`, the trait `Unpin` is not implemented for `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>`
   |
   = note: required because it appears within the type `impl Future`
   = note: required because it appears within the type `async_stream::async_stream::AsyncStream<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 'static)>>, impl Future>`
   = note: required because it appears within the type `impl Stream`
   = note: required because it appears within the type `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because it appears within the type `tokio_stream::map::_::__Origin<'_, tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because it appears within the type `tokio_stream::take::_::__Origin<'_, tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::take::Take<tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
```

如果您遇到这样的错误消息，请尝试 pin 住值！

在尝试运行此操作之前，请启动Mini-Redis服务器：

```shell
mini-redis-server
```

然后尝试运行代码。我们将看到输出到 STDOUT 的消息。

```shell
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"four" })
got = Ok(Message { channel: "numbers", content: b"five" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

由于订阅和发布之间存在竞争，因此可能会删除一些早期消息。该程序永远不会退出。只要服务器处于活动状态，对 Mini-Redis 通道的订阅就保持活跃。

让我们看看如何利用 Stream 来扩展该程序。

## Adapters

使用 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) 作为参数并且返回另一个 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) 的函数通常被成为 “Stream 适配器”，因为它们是“适配器模式”的形式。公共 Stream 适配器包括 [`map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map)，[`take`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.take) 和 [`filter`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter)。

让我们更新 Mini-Redis 以便它退出。收到三条消息后，停止迭代消息。这是使用 [`take`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.take) 完成的。此适配器限制 stream 最多产生 `n` 条消息。

```rust
let messages = subscriber
    .into_stream()
    .take(3);
```

再次运行程序，我们得到：

```shell
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
```

这次程序结束了。

现在，让我们将 Stream 数限制为的单个数字。我们将通过检查消息长度来检查。我们使用 [`filter`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter) 适配器删除与谓词不匹配的任何消息。

```rust
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .take(3);
```

再次运行程序，我们得到：

```shell
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

请注意，适配器的应用顺序很重要。先调用 `filter` 然后 `take` 与先调用 `take` 然后 `filter` 不同。

最后，我们将通过剥离 `Ok(Message { ... })` 的一部分来整理输出。这是通过 [`map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map) 完成的。因为这是在 `filter` 之后应用的，所以我们知道消息 `Ok`，因此我们可以使用 `unwrap()`。

```rust
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
        Ok(msg) if msg.content.len() == 1 => true,
        _ => false,
    })
    .map(|msg| msg.unwrap().content)
    .take(3);
```

现在，输出为：

```shell
got = b"1"
got = b"3"
got = b"6"
```

另一个选项是将 [`filter`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter) 组合 [`map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.map) 到使用 [`filter_map`](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html#method.filter_map) 单个调用中。

还有更多可用的适配器。请参阅[此处](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html)的列表。

## Implementing `Stream`

[`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) trait 与 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 非常相似。

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;

    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

`Stream::poll_next()` 函数非常类似于 `Future::poll`，但可以重复调用它以从 Stream 中接收许多值。正如我们在 [Async in depth](./async_in_depth.md) 中看到的那样，当 Stream 还没有准备好返回值时，返回 `Poll::Pending`。该任务的 Waker 将被注册。一旦需要再次轮询该 Stream，就会通知 Waker。

`size_hint()` 方法的使用方式与[迭代器](https://doc.rust-lang.org/book/ch13-02-iterators.html)相同。

通常，当手动实现 `Stream` 时，它是通过组成 Future 和其他 Stream 来完成的。让我们以 [Async in depth](./async_in_depth.md) 中实现的 `Delay` Future 来举例。我们将其转换为以 10 毫秒为间隔产生三次 () 的 Stream。

```rust
use tokio_stream::Stream;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

struct Interval {
    rem: usize,
    delay: Delay,
}

impl Interval {
    fn new() -> Self {
        Self {
            rem: 3,
            delay: Delay { when: Instant::now() }
        }
    }
}

impl Stream for Interval {
    type Item = ();

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<()>>
    {
        if self.rem == 0 {
            // No more delays
            return Poll::Ready(None);
        }

        match Pin::new(&mut self.delay).poll(cx) {
            Poll::Ready(_) => {
                let when = self.delay.when + Duration::from_millis(10);
                self.delay = Delay { when };
                self.rem -= 1;
                Poll::Ready(Some(()))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}
```

### async-stream

使用 [`Stream`](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) trait 手动实现 Stream 很乏味。不幸的是，Rust 编程语言尚未支持 `async/await` 定义 Stream 的语法。正在进行，但尚未准备好。

[`async-stream`](https://docs.rs/async-stream) crate 可作为临时解决方案使用。这个 crate 提供了一个 `stream!` 宏将输入转换为 Stream。使用此 crate，可以这样实现上述间隔：

```rust
use async_stream::stream;
use std::time::{Duration, Instant};

stream! {
    let mut when = Instant::now();
    for _ in 0..3 {
        let delay = Delay { when };
        delay.await;
        yield ();
        when += Duration::from_millis(10);
    }
}
```
