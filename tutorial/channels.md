# Channels

现在我们已经了解了一些关于 Tokio 并发的知识，让我们将其应用到客户端。将我们之前编写的服务器代码放入二进制文件中：

```shell
mkdir src/bin
mv src/main.rs src/bin/server.rs
```

并创建一个新的二进制文件，其中将包含客户端代码：

```shell
touch src/bin/client.rs
```

在此文件中，您将编写此页面的代码。每当您想要运行它时，您都必须首先在单独的终端窗口中启动服务器：

```shell
cargo run --bin server
```

然后是客户端：

```shell
cargo run --bin client
```

话虽这么说，让我们编码吧！

假设我们要运行两个并发的 Redis 命令。我们可以为每个命令生成一个任务。然后这两个命令将同时发生。

首先，我们可能会尝试这样：

```rust
use mini_redis::client;

#[tokio::main]
async fn main() {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Spawn two tasks, one gets a key, the other sets a key
    let t1 = tokio::spawn(async {
        let res = client.get("foo").await;
    });

    let t2 = tokio::spawn(async {
        client.set("foo", "bar".into()).await;
    });

    t1.await.unwrap();
    t2.await.unwrap();
}
```

这不会编译，因为这两个任务都需要以某种方式访问 `client`。由于 `Client` 没有实现 `Copy`，因此如果没有一些代码来促进这种共享，它将无法编译。此外， `Client::set` 采用 `&mut self`，这意味着调用它需要独占访问权限。我们可以为每个任务打开一个连接，但这并不理想。我们不能使用 `std::sync::Mutex`，因为可能在持有锁的时候调用 `.await`。我们可以使用 `tokio::sync::Mutex`， 但这只允许一个正在处理的请求。如果客户端实现了[流水线](https://redis.io/topics/pipelining)，async mutex 会导致连接利用率不足。

## 消息传递

答案是使用消息传递。该模式涉及生成一个专用任务来管理 `client` 资源。任何希望发出请求的任务都会向 `client` 任务发送消息。 `client` 任务代表发送者发出请求，并将响应返回给发送者。

## Tokio 的 Channel 原语

Tokio 提供了许多 [channel](https://docs.rs/tokio/1/tokio/sync/index.html)，每个 channel 都有不同的用途。

- [mpsc](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html)：多生产者、单消费者通道。可以发送许多值。
- [oneshot](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html)：单一生产者，单一消费者渠道。可以发送单个值。
- [broadcast](https://docs.rs/tokio/1/tokio/sync/broadcast/index.html)：多生产者，多消费者。可以发送许多值。每个接收者都会看到每个值。
- [watch](https://docs.rs/tokio/1/tokio/sync/watch/index.html)：多生产者，多消费者。可以发送许多值，但不会保留历史记录。接收者只能看到最新的值。

如果您需要一个多生产者多消费者通道，其中只有一个消费者看到每条消息，则可以使用 [async-channel](https://docs.rs/async-channel/) crate。还有一些可以在异步 Rust 之外使用的通道，例如 [std::sync::mpsc](https://doc.rust-lang.org/stable/std/sync/mpsc/index.html) 和 [crossbeam::channel](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html)。这些通道通过阻塞线程来等待消息，这在异步代码中是不允许的。

在本节中，我们将使用 [mpsc](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html) 和 [oneshot](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html)。其他类型的消息传递通道将在后面的部分中探讨。本节的完整代码可在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs)找到。

## 定义消息类型

在大多数情况下，当使用消息传递时，接收消息的任务会响应多个命令。在我们的例子中，任务将响应 `GET` 和 `SET` 命令。为了对此进行建模，我们首先定义一个 `Command` 枚举并包含每个命令类型的变体。

```rust
use bytes::Bytes;

#[derive(Debug)]
enum Command {
    Get {
        key: String,
    },
    Set {
        key: String,
        val: Bytes,
    }
}
```

## 创建 Channel

在 `main` 函数中，创建一个 `mpsc` channel。

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Create a new channel with a capacity of at most 32.
    let (tx, mut rx) = mpsc::channel(32);

    // ... Rest comes here
}
```

`mpsc` 通道用于向管理 `redis` 连接的任务发送命令。多生产者功能允许多个任务发送消息。创建通道会返回两个值：发送者和接收者。两个句柄分开使用。他们可能会被转移到不同的任务。

创建的通道容量为 32。如果消息发送速度快于接收速度，通道将存储它们。一旦 32 条消息存储在通道中，调用 `send(...).await` 将进入休眠状态，直到接收者删除一条消息。

多个任务发送是通过复制 Sender 来完成的。例如：

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("sending from first handle").await.unwrap();
    });

    tokio::spawn(async move {
        tx2.send("sending from second handle").await.unwrap();
    });

    while let Some(message) = rx.recv().await {
        println!("GOT = {}", message);
    }
}
```

两条消息都发送到单个 `Receiver` 句柄。`mpsc` 通道的 `Receiver` 无法被复制。

当每个 `Sender` 超出作用域或以其他方式被删除时，它不再可能向通道发送更多消息。此时，对 `Receiver` 调用 `recv` 将返回 `None`，这意味着所有发送方都消失了并且通道已关闭。

在我们的管理 Redis 连接的任务的例子中，它知道一旦通道关闭，它就可以关闭 Redis 连接，因为该连接将不会再次使用。

## 生成管理任务

接下来，生成一个处理来自通道的消息的任务。首先，建立与 Redis 的客户端连接。然后，接收到的命令通过 Redis 连接发出。

```rust
use mini_redis::client;
// The `move` keyword is used to **move** ownership of `rx` into the task.
let manager = tokio::spawn(async move {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Start receiving messages
    while let Some(cmd) = rx.recv().await {
        use Command::*;

        match cmd {
            Get { key } => {
                client.get(&key).await;
            }
            Set { key, val } => {
                client.set(&key, val).await;
            }
        }
    }
});
```

现在，更新这两个任务以通过通道发送命令，而不是直接在 Redis 连接上发出命令。

```rust
// The `Sender` handles are moved into the tasks. As there are two
// tasks, we need a second `Sender`.
let tx2 = tx.clone();

// Spawn two tasks, one gets a key, the other sets a key
let t1 = tokio::spawn(async move {
    let cmd = Command::Get {
        key: "foo".to_string(),
    };

    tx.send(cmd).await.unwrap();
});

let t2 = tokio::spawn(async move {
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
    };

    tx2.send(cmd).await.unwrap();
});
```

在 `main` 函数的结尾，我们 `.await` 连接句柄以确保命令在进程退出之前完全完成。

```rust
t1.await.unwrap();
t2.await.unwrap();
manager.await.unwrap();
```

## 接收响应

最后一步是接收来自管理任务返回的响应。`GET` 命令需要获取值，`SET` 命令需要知道操作是否成功完成。

为了传递响应，使用了 `oneshot` 通道。 `oneshot` 通道是一个单生产者、单消费者通道，针对发送单个值进行了优化。在我们的例子中，单一值就是响应。

与 `mpsc` 类似，`oneshot::channel()` 返回发送者和接收者句柄。

```rust
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel();
```

与 `mpsc` 不同，没有指定容量，因为容量始终为 1。此外，这两个句柄都无法复制。

要从接收管理任务的响应，需要在发送命令之前创建好 `oneshot` 通道。通道的 Sender 部分包含在管理任务的命令中。接收部分用于接收响应。

首先，更新 `Command` 以包含 `Sender`。为了方便起见，使用类型别名来引用 `Sender`。

```rust
use tokio::sync::oneshot;
use bytes::Bytes;

/// Multiple different commands are multiplexed over a single channel.
#[derive(Debug)]
enum Command {
    Get {
        key: String,
        resp: Responder<Option<Bytes>>,
    },
    Set {
        key: String,
        val: Bytes,
        resp: Responder<()>,
    },
}

/// Provided by the requester and used by the manager task to send
/// the command response back to the requester.
type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
```

现在，更新发出命令的任务以包含 `oneshot::Sender`。

```rust
let t1 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Get {
        key: "foo".to_string(),
        resp: resp_tx,
    };

    // Send the GET request
    tx.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});

let t2 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
        resp: resp_tx,
    };

    // Send the SET request
    tx2.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});
```

最后，更新管理任务以通过 `oneshot` 通道发送响应。

```rust
while let Some(cmd) = rx.recv().await {
    match cmd {
        Command::Get { key, resp } => {
            let res = client.get(&key).await;
            // Ignore errors
            let _ = resp.send(res);
        }
        Command::Set { key, val, resp } => {
            let res = client.set(&key, val).await;
            // Ignore errors
            let _ = resp.send(res);
        }
    }
}
```

在 `oneshot::Sender` 上调用 `send` 会立即完成，不需要 `.await`。这是因为在 `oneshot` 通道上 `send` 总是会立即失败或成功，而无需任何形式的等待。

当接收部分掉线时，在 `oneshot` 通道上发送值会返回 `Err`。这表明接收者不再对响应感兴趣。在我们的场景中，接收方取消利息是可接受的事件。 `resp.send(...)` 返回的 `Err` 不需要处理。

可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs)找到完整的代码。

## 背压和有界通道

每当引入并发或排队时，确保排队是有界的并且系统能够妥善处理负载非常重要。无界队列最终将填满所有可用内存并导致系统以不可预测的方式失败。

Tokio 会注意避免隐式排队。其中很大一部分原因是异步操作是惰性的。考虑以下几点：

```rust
loop {
    async_op();
}
```

如果异步操作急切地运行，则这个循环将重复创建 `async_op` 操作并入队，且不确保先前的操作已完成。这会导致隐式无界排队。基于回调的系统和基于 eager future 的系统特别容易受到此影响。

然而，对于 Tokio 和异步 Rust，上面的代码片段根本不会导致 `async_op` 运行。这是因为 `.await` 从未被调用。如果代码片段更新为使用 `.await`，则循环会在重新开始之前等待操作完成。

```rust
loop {
    // Will not repeat until `async_op` completes
    async_op().await;
}
```

必须明确引入并发和队列。执行此操作的方法包括：

- `tokio::spawn`
- `select!`
- `join!`
- `mpsc::channel`

这样做时，请注意确保并发总量是有限的。例如，在编写 TCP accept 循环时，请确保打开的套接字总数是有界的。使用 `mpsc::channel` 时，选择可管理的通道容量。具体的界限值将取决于应用。
