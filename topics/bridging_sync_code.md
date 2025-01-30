# Bridging with sync code

在大多数使用 Tokio 的示例中，我们用 `#[tokio::main]` 标记 main 函数并使整个项目异步。

在某些情况下，您可能需要运行一小部分同步代码。有关此信息的更多信息，请参见 [`spawn_blocking`](https://docs.rs/tokio/1/tokio/task/fn.spawn_blocking.html)。

在其他情况下，将应用程序构建为大部分同步，并具有较小或逻辑上不同的异步部分可能会更容易。例如，GUI 应用程序可能希望在主线程上运行 GUI 代码，并在另一个线程上运行 Tokio 运行时。

本页解释了如何将 async/await 隔离到项目的一小部分。

## What `#[tokio::main]` expands to

`#[tokio::main]` 是一个宏，它用非异步主函数替换主函数，该函数启动运行时，然后调用您的代码。例如，这个：

```rust
#[tokio::main]
async fn main() {
    println!("Hello world");
}
```

变成了：

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            println!("Hello world");
        })
}
```

要在我们自己的项目中使用 async/await，我们可以做类似的事情，在适当的情况下，我们利用 [`block_on`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.block_on) 方法来进入异步上下文。

## A synchronous interface to mini-redis

在本节中，我们将介绍如何通过存储 `Runtime` 对象并使用其 `block_on` 方法来构建 mini-redis 的同步接口。在以下各节中，我们将讨论一些替代方法以及何时应使用每种方法。

我们将要包装的接口是异步 [`Client`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html) 类型。它有几种方法，我们将实现以下方法的阻止版本：

- [`Client::get`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html#method.get)
- [`Client::set`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html#method.set)
- [`Client::set_expires`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html#method.set_expires)
- [`Client::publish`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html#method.publish)
- [`Client::subscribe`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html#method.subscribe)

为此，我们引入一个名为 `src/clients/blocking_client.rs` 的新文件，并使用异步 `Client` 类型的包装器结构对其进行初始化：

```rust
use tokio::net::ToSocketAddrs;
use tokio::runtime::Runtime;

pub use crate::clients::client::Message;

/// Established connection with a Redis server.
pub struct BlockingClient {
    /// The asynchronous `Client`.
    inner: crate::clients::Client,

    /// A `current_thread` runtime for executing operations on the
    /// asynchronous client in a blocking manner.
    rt: Runtime,
}

impl BlockingClient {
    pub fn connect<T: ToSocketAddrs>(addr: T) -> crate::Result<BlockingClient> {
        let rt = tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()?;

        // Call the asynchronous connect method using the runtime.
        let inner = rt.block_on(crate::clients::Client::connect(addr))?;

        Ok(BlockingClient { inner, rt })
    }
}
```

在这里，我们包含了构造函数作为如何在非异步上下文中执行异步方法的第一个示例。我们使用 Tokio [`Runtime`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html) 类型上的 [`block_on`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.block_on) 方法执行此操作，该方法执行异步方法并返回其结果。

一个重要的细节是 [`current_thread`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread) 运行时的使用。通常使用 Tokio 时，您将使用默认的 [`multi_thread`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_multi_thread) 运行时，它将生成一堆后台线程，以便它可以同时高效地运行许多任务。对于我们的用例，我们一次只做一件事，因此运行多个线程不会有任何好处。这使得 [`current_thread`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread) 运行时成为完美的选择，因为它不会生成任何线程。

[`enable_all`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.enable_all) 调用启用 Tokio 运行时上的 IO 和计时器驱动程序。如果未启用它们，则运行时无法执行 IO 或计时器。

> 由于 `current_thread` 运行时不会生成线程，因此它仅在调用 `block_on` 时运行。一旦 `block_on` 返回，该运行时上生成的所有任务都将冻结，直到您再次调用 `block_on`。

一旦拥有此结构，大多数方法就很容易实现：

```rust
use bytes::Bytes;
use std::time::Duration;

impl BlockingClient {
    pub fn get(&mut self, key: &str) -> crate::Result<Option<Bytes>> {
        self.rt.block_on(self.inner.get(key))
    }

    pub fn set(&mut self, key: &str, value: Bytes) -> crate::Result<()> {
        self.rt.block_on(self.inner.set(key, value))
    }

    pub fn set_expires(
        &mut self,
        key: &str,
        value: Bytes,
        expiration: Duration,
    ) -> crate::Result<()> {
        self.rt.block_on(self.inner.set_expires(key, value, expiration))
    }

    pub fn publish(&mut self, channel: &str, message: Bytes) -> crate::Result<u64> {
        self.rt.block_on(self.inner.publish(channel, message))
    }
}
```

[`Client::subscribe`](https://docs.rs/mini-redis/0.4/mini_redis/client/struct.Client.html#method.subscribe) 方法更有趣，因为它将 `Client` 转换为 `Subscriber` 对象。我们可以按以下方式实现它：

```rust
/// A client that has entered pub/sub mode.
///
/// Once clients subscribe to a channel, they may only perform
/// pub/sub related commands. The `BlockingClient` type is
/// transitioned to a `BlockingSubscriber` type in order to
/// prevent non-pub/sub methods from being called.
pub struct BlockingSubscriber {
    /// The asynchronous `Subscriber`.
    inner: crate::clients::Subscriber,

    /// A `current_thread` runtime for executing operations on the
    /// asynchronous client in a blocking manner.
    rt: Runtime,
}

impl BlockingClient {
    pub fn subscribe(self, channels: Vec<String>) -> crate::Result<BlockingSubscriber> {
        let subscriber = self.rt.block_on(self.inner.subscribe(channels))?;
        Ok(BlockingSubscriber {
            inner: subscriber,
            rt: self.rt,
        })
    }
}

impl BlockingSubscriber {
    pub fn get_subscribed(&self) -> &[String] {
        self.inner.get_subscribed()
    }

    pub fn next_message(&mut self) -> crate::Result<Option<Message>> {
        self.rt.block_on(self.inner.next_message())
    }

    pub fn subscribe(&mut self, channels: &[String]) -> crate::Result<()> {
        self.rt.block_on(self.inner.subscribe(channels))
    }

    pub fn unsubscribe(&mut self, channels: &[String]) -> crate::Result<()> {
        self.rt.block_on(self.inner.unsubscribe(channels))
    }
}
```

所以 `subscribe` 方法会先通过 Runtime 将异步的 `Client` 转化为异步的 `Subscriber`，然后把得到的 `Subscriber` 和 `Runtime` 一起存储起来，使用 [`block_on`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.block_on) 实现各种方法。

请注意，异步的 `Subscriber` 结构体有一个非异步方法，名为 `get_subscribed`。为了处理这个问题，我们只需直接调用它，而不涉及运行时。

## Other approaches

上面的部分解释了实现同步包装器的最简单方法，但这不是唯一的方法。方法如下：

- 创建一个 [`runtime`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html) 并对异步代码调用 [`block_on`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.block_on)。
- 创建一个 [`runtime`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html) 并在其上 [`spawn`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.spawn)。
- 在单独的线程中运行 [`runtime`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html) 并向其发送消息。

### Spawning things on a runtime

[`Runtime`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html) 对象具有 [`spawn`](https://docs.rs/tokio/1/tokio/runtime/struct.Runtime.html#method.spawn) 方法。调用此方法时，您可以创建一个新的后台任务在 runtime 上运行。例如：

```rust
use tokio::runtime::Builder;
use tokio::time::{sleep, Duration};

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(1)
        .enable_all()
        .build()
        .unwrap();

    let mut handles = Vec::with_capacity(10);
    for i in 0..10 {
        handles.push(runtime.spawn(my_bg_task(i)));
    }

    // Do something time-consuming while the background tasks execute.
    std::thread::sleep(Duration::from_millis(750));
    println!("Finished time-consuming task.");

    // Wait for all of them to complete.
    for handle in handles {
        // The `spawn` method returns a `JoinHandle`. A `JoinHandle` is
        // a future, so we can wait for it using `block_on`.
        runtime.block_on(handle).unwrap();
    }
}

async fn my_bg_task(i: u64) {
    // By subtracting, the tasks with larger values of i sleep for a
    // shorter duration.
    let millis = 1000 - 50 * i;
    println!("Task {} sleeping for {} ms.", i, millis);

    sleep(Duration::from_millis(millis)).await;

    println!("Task {} stopping.", i);
}
```

```shell
Task 0 sleeping for 1000 ms.
Task 1 sleeping for 950 ms.
Task 2 sleeping for 900 ms.
Task 3 sleeping for 850 ms.
Task 4 sleeping for 800 ms.
Task 5 sleeping for 750 ms.
Task 6 sleeping for 700 ms.
Task 7 sleeping for 650 ms.
Task 8 sleeping for 600 ms.
Task 9 sleeping for 550 ms.
Task 9 stopping.
Task 8 stopping.
Task 7 stopping.
Task 6 stopping.
Finished time-consuming task.
Task 5 stopping.
Task 4 stopping.
Task 3 stopping.
Task 2 stopping.
Task 1 stopping.
Task 0 stopping.
```

在上面的例子中，我们在运行时生成 10 个后台任务，然后等待它们所有完成。举例来说，这可能是在图形应用程序中实现后台网络请求的好方法，因为网络请求太耗时，无法在主 GUI 线程上运行它们。相反，您在后台运行的 Tokio 运行时生成请求，并让任务在请求完成时将信息发送回 GUI 代码，或者如果您想要进度条，甚至可以逐步发送信息。

在这个例子中，重要的是将运行时配置为 [`multi_thread`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_multi_thread)。如果将其更改为 [`current_thread`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread) 运行时，您会发现耗时任务在任何后台任务启动之前就完成。这是因为在 `current_thread` 运行时产生的后台任务只会在调用 `block_on` 期间执行，否则运行时就没有地方运行它们。

该示例通过调用 spawn 调用返回的 [`JoinHandle`](https://docs.rs/tokio/1/tokio/task/struct.JoinHandle.html) 上的 `block_on` 来等待生成的任务完成，但这不是唯一的方法。以下是一些替代方案：

- 使用消息传递通道，例如 [`tokio::sync::mpsc`](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html)。
- 修改受互斥锁等保护的共享值。对于 GUI 中的进度条来说，这可能是一种很好的方法，因为 GUI 每帧都会读取共享值。

[`Handle`](https://docs.rs/tokio/1/tokio/runtime/struct.Handle.html) 类型也提供 `spawn` 方法。可以复制 `Handle` 类型以获取运行时的多个 handles，并且每个 `Handle` 都可用于在运行时上生成新任务。

### Sending messages

第三种技术是生成一个运行时并使用消息传递与其通信。这比其他两种方法涉及更多的样板，但它是最灵活的方法。您可以在下面找到一个基本示例：

```rust
use tokio::runtime::Builder;
use tokio::sync::mpsc;

pub struct Task {
    name: String,
    // info that describes the task
}

async fn handle_task(task: Task) {
    println!("Got task {}", task.name);
}

#[derive(Clone)]
pub struct TaskSpawner {
    spawn: mpsc::Sender<Task>,
}

impl TaskSpawner {
    pub fn new() -> TaskSpawner {
        // Set up a channel for communicating.
        let (send, mut recv) = mpsc::channel(16);

        // Build the runtime for the new thread.
        //
        // The runtime is created before spawning the thread
        // to more cleanly forward errors if the `unwrap()`
        // panics.
        let rt = Builder::new_current_thread()
            .enable_all()
            .build()
            .unwrap();

        std::thread::spawn(move || {
            rt.block_on(async move {
                while let Some(task) = recv.recv().await {
                    tokio::spawn(handle_task(task));
                }

                // Once all senders have gone out of scope,
                // the `.recv()` call returns None and it will
                // exit from the while loop and shut down the
                // thread.
            });
        });

        TaskSpawner {
            spawn: send,
        }
    }

    pub fn spawn_task(&self, task: Task) {
        match self.spawn.blocking_send(task) {
            Ok(()) => {},
            Err(_) => panic!("The shared runtime has shut down."),
        }
    }
}
```

此示例可以以多种方式配置。例如，您可以使用 [`Semaphore`](https://docs.rs/tokio/1/tokio/sync/struct.Semaphore.html) 来限制活动任务的数量，或者您可以使用反方向的通道向生成器发送响应。当您以这种方式生成运行时时，它就是一种 [`actor`](https://ryhl.io/blog/actors-with-tokio/)。
