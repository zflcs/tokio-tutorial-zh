# Async in depth

至此，我们已经完成了对异步 Rust 和 Tokio 的相当全面的浏览。现在我们将深入研究 Rust 的异步运行时模型。在本教程的一开始，我们就暗示异步 Rust 采用了一种独特的方法。现在，我们解释一下这意味着什么。

## Futures

作为快速回顾，我们来看一个非常基本的异步函数。与本教程到目前为止所涵盖的内容相比，这并不是什么新鲜事。

```rust
use tokio::net::TcpStream;

async fn my_async_fn() {
    println!("hello from async");
    let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
    println!("async TCP operation complete");
}
```

我们调用该函数并返回一些值。我们对该值调用 `.await`。

```rust
#[tokio::main]
async fn main() {
    let what_is_this = my_async_fn();
    // Nothing has been printed yet.
    what_is_this.await;
    // Text has been printed and socket has been
    // established and closed.
}
```

`my_async_fn()` 返回的值是一个 future。future 是一个实现标准库提供的 [`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的值。它们是包含正在进行的异步计算的值。

[`std::future::Future`](https://doc.rust-lang.org/std/future/trait.Future.html) trait 的定义是：

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Self::Output>;
}
```

[关联类型](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types) Output 是 future 完成后生成的类型。[`Pin`](https://doc.rust-lang.org/std/pin/index.html) 类型使得 Rust 在 `async` 函数中能够支持借用。有关更多详细信息，请参阅[标准库](https://doc.rust-lang.org/std/pin/index.html)文档。

与其他语言中的 future 实现方式不同，Rust future 并不代表在后台发生的计算，而是代表计算本身。Future 的所有者负责通过 polling future 来推进计算。这是通过调用 `Future::poll` 来完成的。

### 实现 Future

让我们实现一个非常简单的 future。这个 future 将：

1. 等到特定的时刻。
2. 输出一些文本到 `STDOUT`。
3. 产生一个字符串。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let future = Delay { when };

    let out = future.await;
    assert_eq!(out, "done");
}
```

### Async fn as a Future

在 main 函数中，我们实例化 future 并调用 `.await`。在异步函数中，我们可以对任何实现 `Future` 的值调用.await 。反过来，调用 `async` 函数会返回一个实现了 Future 的匿名类型。在 `async fn main()` 的情况下，生成的 future 大致为：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
    // Initialized, never polled
    State0,
    // Waiting on `Delay`, i.e. the `future.await` line.
    State1(Delay),
    // The future has completed.
    Terminated,
}

impl Future for MainFuture {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<()>
    {
        use MainFuture::*;

        loop {
            match *self {
                State0 => {
                    let when = Instant::now() +
                        Duration::from_millis(10);
                    let future = Delay { when };
                    *self = State1(future);
                }
                State1(ref mut my_future) => {
                    match Pin::new(my_future).poll(cx) {
                        Poll::Ready(out) => {
                            assert_eq!(out, "done");
                            *self = Terminated;
                            return Poll::Ready(());
                        }
                        Poll::Pending => {
                            return Poll::Pending;
                        }
                    }
                }
                Terminated => {
                    panic!("future polled after completion")
                }
            }
        }
    }
}
```

Rust future 是**状态机**。这里， `MainFuture` 表示为 future 可能状态的 `enum`。Future 从 `State0` 状态开始。当调用 `poll` 时，future 会尝试尽可能地推进其内部状态。如果 Future 能够完成，则返回 `Poll::Ready` 其中包含异步计算的输出。

如果 future 无法完成，通常是由于它正在等待的资源尚未准备好，则返回 `Poll::Pending`。返回 `Poll::Pending` 向调用者表明 future 将在稍后完成，调用者应稍后再次调用 `poll`。

我们还看到 future 是由其他 future 组成的。对外部 future 调用 `poll` 会导致调用内部 future 的 `poll` 函数。

## Executors

异步 Rust 函数返回 future。Futures 必须有 `poll` 来推进自己的状态。future 是由其他 future 组成的。那么，问题是，对最外层 future 如何调用 `poll` 方法？

回想一下之前的内容，要运行异步函数，它们必须传递给 `tokio::spawn` 或者是带有 `#[tokio::main]` 注释的 main 函数。这导致将生成的外部 future 提交给 Tokio executor。Executor 负责在外部 future 上调用 `Future::poll`，驱动异步计算完成。

### Mini Tokio

为了更好地理解这一切是如何组合在一起的，让我们实现我们自己的最小版本的 Tokio！完整的代码可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/mini-tokio/src/main.rs)找到。

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }

    /// Spawn a future onto the mini-tokio instance.
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }

    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);

        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

这将运行异步块。将会创建一个 `Delay` 实例并等待。然而，我们迄今为止的实现有一个重大缺陷。我们的 executor  从不休眠。executor  不断循环所有产生 future 并对其进行轮询。大多数时候，future 还没有准备好执行更多工作并将再次返回 `Poll::Pending `。该过程会消耗 CPU 周期，并且通常效率不高。

理想情况下，我们希望 mini-tokio 仅在 future 能够取得进展时才轮询 future。当任务被阻塞的资源准备好执行请求的操作时，就会发生这种情况。如果任务想要从 TCP 套接字读取数据，那么我们只想在 TCP 套接字接收到数据时轮询任务。在我们的例子中，任务在到达给定 Instant 之前将被阻止。理想情况下，mini-tokio 只会在该时刻过去后轮询任务。

为了实现这一点，当资源被轮询并且资源尚未准备好时，资源一旦转换为就绪状态就会发送通知。

## Wakers

Wakers 是缺失的一块。这是一个资源能够通知等待任务该资源已准备好继续某些操作的系统。

让我们再次看看 `Future::poll` 定义：

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context)
    -> Poll<Self::Output>;
```

`poll` 的 `Context` 参数有一个 `waker()` 方法。该方法返回一个绑定到当前任务的 [`Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html)。 [`Waker`](https://doc.rust-lang.org/std/task/struct.Waker.html) 有一个 `wake()` 方法。调用此方法向 executor 发出信号，指示应安排执行关联的任务。资源在转换到就绪状态时调用 `wake()`，以通知 executor 轮询任务将能够取得进展。

### 更新 Delay

我们可以更新 `Delay` 以使用 wakers：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;

struct Delay {
    when: Instant,
}

impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Get a handle to the waker for the current task
            let waker = cx.waker().clone();
            let when = self.when;

            // Spawn a timer thread.
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                waker.wake();
            });

            Poll::Pending
        }
    }
}
```

现在，一旦请求的持续时间过去，调用任务就会收到通知，executor  可以确保再次调度该任务。下一步是更新 mini-tokio 以侦听唤醒通知。

我们的 `Delay` 实现仍然存在一些问题。我们稍后会修复它们。

> 当 future 返回 `Poll::Pending` 时，它必须确保 waker 在某个时刻收到信号。忘记执行此操作会导致任务无限期挂起。
>
> 返回 `Poll::Pending` 后忘记唤醒任务是错误的常见来源。

回想一下 `Delay` 的第一次迭代。这是 future 的实现：

```rust
impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

在返回 `Poll::Pending` 之前，我们调用了 `cx.waker().wake_by_ref()`。这是为了满足 future 的 contract。通过返回 `Poll::Pending`，我们负责向 waker 发出信号。因为我们还没有实现计时器线程，所以我们内联地向 waker 发出信号。这样做将导致 future 立即重新调度、再次执行，并且此时可能尚未准备好完成。

请注意，您可以比必要时更频繁地向 waker 发出信号。在这种特殊情况下，即使我们根本没有准备好继续操作，我们也会向 waker 发出信号。除了浪费一些 CPU 周期之外，这并没有什么问题。然而，这种特定的实现将导致繁忙循环。

### Updating Mini Tokio

下一步是更新 Mini Tokio 以接收唤醒通知。我们希望 executor 仅在任务被唤醒时运行任务，为此，Mini Tokio 将提供自己的 waker。当waker被调用时，其关联的任务将排队等待执行。 Mini-Tokio 在对 future 进行轮询时将这个 waker 传递给 future。

更新后的 Mini Tokio 将使用通道来存储计划任务。通道允许任务排队等待从任何线程执行。Wakers 必须实现 `Send` 和 `Sync`。

> Send 和 Sync trait 是 Rust 提供的与并发相关的标记特征。可以发送到不同线程的类型是 `Send`。大多数类型是 `Send`，但像 [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html) 这样的类型不是。可以通过不可变引用并发访问的类型是 `Sync` 。类型可以是 `Send` 但不能是 `Sync` ——一个很好的例子是 [`Cell`](https://doc.rust-lang.org/std/cell/struct.Cell.html)，可以通过不可变引用进行修改，因此并发访问不安全。
>
> 更多详细信息，请参阅[Rust 书籍中的相关章节](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html)。

更新 `MiniTokio` 结构。

```rust
use std::sync::mpsc;
use std::sync::Arc;

struct MiniTokio {
    scheduled: mpsc::Receiver<Arc<Task>>,
    sender: mpsc::Sender<Arc<Task>>,
}

struct Task {
    // This will be filled in soon.
}
```

Waker 是 `Sync` 并且可以被复制。当 `wake` 被调用时，任务必须被安排执行。为了实现这一点，我们有一个通道。当 waker 调用 `wake()`时，任务被推送到通道的发送部分。 我们的 Task 结构将实现唤醒逻辑。为此，它需要包含生成的 future 和通道的发送端。我们将 future 放在 `TaskFuture` 结构中，并与 `Poll` 枚举一起放置，以跟踪最新的情况 `Future::poll()` 结果，需要处理虚假唤醒。更多细节在 `TaskFuture` 中 `poll()` 方法的实现中给出。

```rust
use std::sync::{Arc, Mutex};

/// A structure holding a future and the result of
/// the latest call to its `poll` method.
struct TaskFuture {
    future: Pin<Box<dyn Future<Output = ()> + Send>>,
    poll: Poll<()>,
}

struct Task {
    // The `Mutex` is to make `Task` implement `Sync`. Only
    // one thread accesses `task_future` at any given time.
    // The `Mutex` is not required for correctness. Real Tokio
    // does not use a mutex here, but real Tokio has
    // more lines of code than can fit in a single tutorial
    // page.
    task_future: Mutex<TaskFuture>,
    executor: mpsc::Sender<Arc<Task>>,
}

impl Task {
    fn schedule(self: &Arc<Self>) {
        self.executor.send(self.clone());
    }
}
```

为了调度任务， `Arc` 被复制并通过通道发送。现在，我们需要将 `schedule` 函数与 `std::task::Waker` 关联。标准库提供了一个底层 API，使用[手动 vtable 构建](https://doc.rust-lang.org/std/task/struct.RawWakerVTable.html)来完成此操作。该策略为实现者提供了最大的灵活性，但需要一堆不安全的样板代码。相比于直接使用 [`RawWakerVTable`](https://doc.rust-lang.org/std/task/struct.RawWakerVTable.html)，我们将使用 [`futures`](https://docs.rs/futures/) crate 提供的 [`ArcWake`](https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html) 工具。这允许我们实现一个简单的特征，将我们的 `Task` 结构公开为waker。

将以下依赖项添加到您的 `Cargo.toml` 中以引入 `futures`。

```rust
futures = "0.3"
```

然后实现 [`futures::task::ArcWake`](https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html) 。

```rust
use futures::task::{self, ArcWake};
use std::sync::Arc;
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        arc_self.schedule();
    }
}
```

当上面的定时器线程调用 `waker.wake()` 时，任务被推送到通道。接下来我们实现 `MiniTokio::run()` 函数中的接收并执行任务。

```rust
impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    /// Initialize a new mini-tokio instance.
    fn new() -> MiniTokio {
        let (sender, scheduled) = mpsc::channel();

        MiniTokio { scheduled, sender }
    }

    /// Spawn a future onto the mini-tokio instance.
    ///
    /// The given future is wrapped with the `Task` harness and pushed into the
    /// `scheduled` queue. The future will be executed when `run` is called.
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl TaskFuture {
    fn new(future: impl Future<Output = ()> + Send + 'static) -> TaskFuture {
        TaskFuture {
            future: Box::pin(future),
            poll: Poll::Pending,
        }
    }

    fn poll(&mut self, cx: &mut Context<'_>) {
        // Spurious wake-ups are allowed, even after a future has                                  
        // returned `Ready`. However, polling a future which has                                   
        // already returned `Ready` is *not* allowed. For this                                     
        // reason we need to check that the future is still pending                                
        // before we call it. Failure to do so can lead to a panic.
        if self.poll.is_pending() {
            self.poll = self.future.as_mut().poll(cx);
        }
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        // Create a waker from the `Task` instance. This
        // uses the `ArcWake` impl from above.
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);

        // No other thread ever tries to lock the task_future
        let mut task_future = self.task_future.try_lock().unwrap();

        // Poll the inner future
        task_future.poll(&mut cx);
    }

    // Spawns a new task with the given future.
    //
    // Initializes a new Task harness containing the given future and pushes it
    // onto `sender`. The receiver half of the channel will get the task and
    // execute it.
    fn spawn<F>(future: F, sender: &mpsc::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            task_future: Mutex::new(TaskFuture::new(future)),
            executor: sender.clone(),
        });

        let _ = sender.send(task);
    }
}
```

这里正在发生很多事情。首先，实现了 `MiniTokio::run()`。该函数在循环中运行，从通道接收调度任务。由于任务在被唤醒时被推送到通道中，因此这些任务在执行时能够取得进展。

此外，`MiniTokio::new()` 和 `MiniTokio::spawn()` 函数被调整为使用通道而不是 `VecDeque`。当新任务产生时，它们会获得通道发送者部分的副本，任务可以使用它在运行时让自己重调度。

`Task::poll()` 函数使用 `futures` crate 中的 [`ArcWake`](https://docs.rs/futures/0.3/futures/task/trait.ArcWake.html) 工具创建 waker。waker用于创建 `task::Context`。而 `task::Context` 被传递给 `poll`。

## 概括

我们现在已经看到了异步 Rust 如何工作的端到端示例。Rust 的 `async/await` 特性由 trait 支持。这允许第三方 crate（例如 Tokio）提供执行的详细信息。

- 异步 Rust 操作是惰性的，需要调用者轮询它们。
- Wakers 被传递给 future，将 future 与调用它的任务联系起来。
- 当资源尚未准备好完成操作时，将返回Poll::Pending并记录任务的唤醒程序。
- 当资源准备就绪时，任务的唤醒程序会收到通知。
- Executor 接收通知并安排任务执行。
- 再次轮询任务，这次资源已准备好并且任务取得进展。

## 一些未解决的问题

回想一下，当我们实现 `Delay` future 时，我们说过还有一些问题需要解决。 Rust 的异步模型允许单个 future 在执行时跨任务迁移。考虑以下几点：

```rust
use futures::future::poll_fn;
use std::future::Future;
use std::pin::Pin;

#[tokio::main]
async fn main() {
    let when = Instant::now() + Duration::from_millis(10);
    let mut delay = Some(Delay { when });

    poll_fn(move |cx| {
        let mut delay = delay.take().unwrap();
        let res = Pin::new(&mut delay).poll(cx);
        assert!(res.is_pending());
        tokio::spawn(async move {
            delay.await;
        });

        Poll::Ready(())
    }).await;
}
```

`poll_fn` 函数使用闭包创建一个 `Future` 实例。上面的代码片段创建了一个 `Delay` 实例，轮询一次，然后将 `Delay` 实例发送到等待它的新任务。在此示例中，使用不同的 `Waker` 实例多次调用 `Delay::poll`。发生这种情况时，您必须确保调用 `wake` 的 `waker` 是最近一次传递给 `poll` 的。

在实现 future 时，假设每次调用 `poll` 都可以提供不同的 `Waker` 实例，这一点至关重要。poll 函数必须用新的 waker 更新任何先前记录的 waker。

我们早期的 `Delay` 实现在每次轮询时都会产生一个新线程。这很好，但如果轮询太频繁，效率可能会非常低（例如，如果您 `select!` 该 future 和其他一些 future，那么当某个时间发生时，每个 future 都会被轮询）。解决这个问题的一种方法是记住您是否已经生成了一个线程，如果还没有生成一个新线程则产生一个。但是，如果您这样做，则必须确保线程的 `Waker` 会在稍后调用 poll 时更新，否则您不会唤醒最近的 `Waker`。

为了修复我们之前的实现，我们可以这样做：

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
    // This is Some when we have spawned a thread, and None otherwise.
    waker: Option<Arc<Mutex<Waker>>>,
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // Check the current instant. If the duration has elapsed, then
        // this future has completed so we return `Poll::Ready`.
        if Instant::now() >= self.when {
            return Poll::Ready(());
        }

        // The duration has not elapsed. If this is the first time the future
        // is called, spawn the timer thread. If the timer thread is already
        // running, ensure the stored `Waker` matches the current task's waker.
        if let Some(waker) = &self.waker {
            let mut waker = waker.lock().unwrap();

            // Check if the stored waker matches the current task's waker.
            // This is necessary as the `Delay` future instance may move to
            // a different task between calls to `poll`. If this happens, the
            // waker contained by the given `Context` will differ and we
            // must update our stored waker to reflect this change.
            if !waker.will_wake(cx.waker()) {
                *waker = cx.waker().clone();
            }
        } else {
            let when = self.when;
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());

            // This is the first time `poll` is called, spawn the timer thread.
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                // The duration has elapsed. Notify the caller by invoking
                // the waker.
                let waker = waker.lock().unwrap();
                waker.wake_by_ref();
            });
        }

        // By now, the waker is stored and the timer thread is started.
        // The duration has not elapsed (recall that we checked for this
        // first thing), ergo the future has not completed so we must
        // return `Poll::Pending`.
        //
        // The `Future` trait contract requires that when `Pending` is
        // returned, the future ensures that the given waker is signalled
        // once the future should be polled again. In our case, by
        // returning `Pending` here, we are promising that we will
        // invoke the given waker included in the `Context` argument
        // once the requested duration has elapsed. We ensure this by
        // spawning the timer thread above.
        //
        // If we forget to invoke the waker, the task will hang
        // indefinitely.
        Poll::Pending
    }
}
```

这有点复杂，但其想法是，在每次调用 `poll` 时，future 都会检查提供的 waker 是否与之前记录的 waker 匹配。如果两个 waker 匹配，则无需执行其他操作。如果它们不匹配，则必须更新记录的 waker。

### Notify 工具

我们演示了如何使用 waker 手动实现 `Delay` future。`Wakers` 是异步 Rust 工作方式的基础。通常情况下，没有必要深入到这个水平。例如，在 `Delay` 的例子中，我们可以通过对 [`tokio::sync::Notify` 工具](https://docs.rs/tokio/1/tokio/sync/struct.Notify.html) 使用 `async/await` 来完全实现它。该工具提供了基本的任务通知机制。它处理 waker 的详细信息，包括确保记录的 waker 与当前任务匹配。

使用 [`Notify`](https://docs.rs/tokio/1/tokio/sync/struct.Notify.html)，我们可以使用 `async/await` 像这样实现 delay 功能：

```rust
use tokio::sync::Notify;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::thread;

async fn delay(dur: Duration) {
    let when = Instant::now() + dur;
    let notify = Arc::new(Notify::new());
    let notify_clone = notify.clone();

    thread::spawn(move || {
        let now = Instant::now();

        if now < when {
            thread::sleep(when - now);
        }

        notify_clone.notify_one();
    });


    notify.notified().await;
}
```
