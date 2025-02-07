# Glossary

## Asynchronous

在 Rust 的上下文中，异步代码是指使用 async/await 语言特性的代码，它允许许多任务在几个线程（甚至单个线程）上同时运行。

## Concurrency and parallelism

并发和并行性是两个相关的概念，在谈论执行多个任务时都可以使用。如果某事并行发生，那么它也同时发生，但相反的情况并非如此：在两个任务之间交替，但从未同时处理这两个任务是并发，但不是并行性。

## Future

Future 是存储某些操作的当前状态的值。Future 也有一种 `poll` 方法，该方法可以使操作继续进行，直到需要等待网络连接之类的东西。对 `poll` 方法的调用应很快返回。

Future 通常是通过在异步块中使用 `.await` 组合多个 Future 来创建的。

## Executor/scheduler

Executor 或者 Scheduler 是通过重复调用 `poll` 方法来执行 Future 的。标准库中没有 Executor，因此您需要一个外部库，而最广泛使用的 Executor 由 Tokio  Runtime 提供。

Executor 能够在几个线程上同时运行大量的 Future。它通过在 await 处交换当前正在运行的任务来实现这一点。如果代码花费很长时间而未到达 `.await`，则称为 "blocking the thread" 或 "not yielding back to the executor"，这会阻止其他任务运行。

## Runtime

Runtime 是一个包含 Executor 以及与该 Executor 集成的各种工具（如计时工具和 IO）的库。 Runtime 和执行器这两个词有时可以互换使用。标准库没有 Runtime ，因此您需要一个外部库，而最广泛使用的 Runtime 是 Tokio Runtime 。

运行时 (Runtime) 一词也用于其他语境，例如，短语“Rust 没有运行时”有时用来表示 Rust 不执行垃圾收集或即时编译。

## Task

任务是在 Tokio Runtime 中运行的操作，由 [`tokio::spawn`](https://docs.rs/tokio/1/tokio/fn.spawn.html) 或 [`Runtime::block_on`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.block_on) 函数创建。通过组合来创建 Future 的工具（例如 `.await` 和 [`join!`](https://docs.rs/tokio/1/tokio/macro.join.html)）不会创建新任务，并且每个组合的部分都被称为“在同一个任务中”。

并行需要多个任务，但可以使用 `join!` 等工具在一个任务上同时执行多项操作。

## Spawning

Spawning 是指使用 [`tokio::spawn`](https://docs.rs/tokio/1/tokio/fn.spawn.html) 函数创建新任务。它也可以指使用 [`std::thread::spawn`](https://doc.rust-lang.org/stable/std/thread/fn.spawn.html) 创建新线程。

## Async block

异步块是创建运行某些代码的 Future 的一种简单方法。例如：

```rust
let world = async {
    println!(" world!");
};
let my_future = async {
    print!("Hello ");
    world.await;
};
```

上面的代码创建了一个名为 `my_future` 的 Future，如果执行则会打印 `Hello world!`。它通过首先打印 hello，然后运行 `​​world` future 来实现这一点。请注意，上述代码不会自行打印任何内容——您必须在任何事情发生之前实际执行 `my_future`，方法是直接生成它，或者在您 spawn 的东西中通过 `.await`ing 它。

## Async function

与异步块类似，异步函数是一种创建函数主体成为 Future 的简单方法。所有异步函数都可以重写为返回 Future 的普通函数：

```rust
async fn do_stuff(i: i32) -> String {
    // do stuff
    format!("The integer is {}.", i)
}
```

```rust
use std::future::Future;

// the async function above is the same as this:
fn do_stuff(i: i32) -> impl Future<Output = String> {
    async move {
        // do stuff
        format!("The integer is {}.", i)
    }
}
```

这使用 [`impl Trait` 语法](https://doc.rust-lang.org/book/ch10-02-traits.html#returning-types-that-implement-traits)来返回 Future，因为 [`Future`](https://doc.rust-lang.org/stable/std/future/trait.Future.html) 是一个 trait。请注意，由于异步块创建的 Future 在执行之前不会执行任何操作，因此调用异步函数在其返回的 Future 执行之前不会执行任何操作[（忽略它会触发警告）](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4faf44e08b4a3bb1269a7985460f1923)。

## Yielding

在异步 Rust 的上下文中，yielding 允许 Executor 在单个线程上执行许多 Future。每当一个 Future yield 时，Executor 就能够将该 Future 与其他 Future 进行交换，并且通过反复交换当前任务，Executor 可以同时执行大量任务。Future 只能在 `.await` 处 yield，因此在 `.await` 之间花费很长时间的 Future 可能会阻止其他任务运行。

具体来说，只要 Future 从 [`poll`](https://doc.rust-lang.org/stable/std/future/trait.Future.html#method.poll) 方法返回，它就会 yield。

## Blocking

"Blocking" 这个词有两种不同的用法："Blocking" 的第一个含义只是等待某件事完成，而 "Blocking" 的另一个含义是 Future 花费很长时间而没有 yield。为了不含糊，您可以使用短语 "blocking the thread" 来表示第二个含义。

Tokio 的文档将始终使用 "Blocking" 的第二个含义。

要在 Tokio 中运行阻塞代码，请参阅 Tokio API 参考中的 [CPU-bound tasks 和 blocking code](https://docs.rs/tokio/1/tokio/#cpu-bound-tasks-and-blocking-code) 部分。

## Stream

[`Stream`](https://docs.rs/tokio-stream/0.1/tokio/trait.Stream.html) 是 [`Iterator`](https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html) 的异步版本，提供值 stream。它通常与 `while let` 循环一起使用，如下所示：

```rust
use tokio_stream::StreamExt; // for next()

while let Some(item) = stream.next().await {
    // do something
}
```

有时候，容易引起混淆的是，"stream" 这个词被用来指代 [`AsyncRead`](https://docs.rs/tokio/1/tokio/io/trait.AsyncRead.html) 和 [`AsyncWrite`](https://docs.rs/tokio/1/tokio/io/trait.AsyncWrite.html) 特征。

Tokio 的 stream 工具目前由 [`tokio-stream`](https://docs.rs/tokio-stream) crate 提供。一旦 `Stream` trait 在 std 中稳定下来，stream 工具将被移入 `tokio` crate。

## Channel

Channel 是一种允许代码的一部分向其他部分发送消息的工具。Tokio 提供了[许多 channel](https://docs.rs/tokio/1/tokio/sync/index.html)，每个 channel 都有不同的用途。

- [mpsc](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html)：多生产者、单消费者通道。可以发送多个值。
- [oneshot](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html)：单生产者，单消费者通道。可以发送单个值。
- [broadcast](https://docs.rs/tokio/1/tokio/sync/broadcast/index.html)：多生产者，多消费者。可以发送多个值。每个接收者都可以看到每个值。
- [watch](https://docs.rs/tokio/1/tokio/sync/watch/index.html)：单生产者，多消费者。可以发送多个值，但不保留历史记录。接收者只能看到最新的值。

如果您需要一个多生产者多消费者 channel，其中只有一个消费者可以看到每条消息，那么您可以使用 [async-channel](https://docs.rs/async-channel/) crate。

还有一些 channel 可以在异步 Rust 之外使用，例如 [`std::sync::mpsc`](https://doc.rust-lang.org/stable/std/sync/mpsc/index.html) 和 [`crossbeam::channel`](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html)。这些 channel 通过阻塞线程来等待消息，这在异步代码中是不允许的。

## Backpressure

背压是一种设计能够很好地响应高负载的应用程序的模式。例如，mpsc 通道有有界和无界两种形式。通过使用有界 channel，当接收方无法跟上消息数量时，接收方可以对发送方施加 "backpressure"，从而避免随着 channel 上发送的消息越来越多而导致内存使用量无限制增长。

## Actor

一种设计应用程序的设计模式。Actor 是指独立生成的任务，它代表应用程序的其他部分管理一些资源，并使用 channel 与应用程序的其他部分进行通信。

请参阅 [channel 章节](https://tokio.rs/tokio/tutorial/channels) 以了解 Actor 的示例。
