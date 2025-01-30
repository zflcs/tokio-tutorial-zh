# Graceful Shutdown

本页的目的是概述如何在异步应用程序中正确实现关闭。

实现优雅关机通常包括三个部分：

- 确定何时关闭。
- 告诉程序的每个部分关闭。
- 等待程序的其他部分关闭。

本文的其余部分将介绍这些部分。这里描述的方法的实际实现可以在 [mini-redis](https://github.com/tokio-rs/mini-redis/) 中找到，具体来说是 [`src/server.rs`](https://github.com/tokio-rs/mini-redis/blob/master/src/server.rs) 和 [`src/shutdown.rs`](https://github.com/tokio-rs/mini-redis/blob/master/src/shutdown.rs) 文件。

## Figuring out when to shut down

这当然取决于应用程序，但一个非常常见的关闭标准是当应用程序接收到来自操作系统的信号时。例如，当程序运行时在终端中按下 ctrl+c 时，就会发生这种情况。为了检测这种情况，Tokio 提供了一个 [`tokio::signal::ctrl_c`](https://docs.rs/tokio/1/tokio/signal/fn.ctrl_c.html) 函数，它将处于休眠状态直到收到这样的信号。您可以像这样使用它：

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    // ... spawn application as separate task ...

    match signal::ctrl_c().await {
        Ok(()) => {},
        Err(err) => {
            eprintln!("Unable to listen for shutdown signal: {}", err);
            // we also shut down in case of error
        },
    }

    // send shutdown signal to application and wait
}
```

如果有多个关闭条件，可以使用 [mpsc 通道](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html) 将关闭信号发送到一个位置。然后，您可以对 [`ctrl_c`](https://docs.rs/tokio/1/tokio/signal/fn.ctrl_c.html) 和通道上使用 [select](https://docs.rs/tokio/1/tokio/macro.select.html)。例如：

```rust
use tokio::signal;
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (shutdown_send, mut shutdown_recv) = mpsc::unbounded_channel();

    // ... spawn application as separate task ...
    //
    // application uses shutdown_send in case a shutdown was issued from inside
    // the application

    tokio::select! {
        _ = signal::ctrl_c() => {},
        _ = shutdown_recv.recv() => {},
    }

    // send shutdown signal to application and wait
}
```

## Telling things to shut down

当你想告诉一个或多个任务关闭时，你可以使用 [Cancellation Tokens](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html)。这些令牌允许您通知任务，它们应该响应取消请求而自行终止，从而轻松实现正常关闭。

要在多个任务之间共享 `CancellationToken`，您必须复制它。这是由于单一所有权规则要求每个值都有一个所有者。复制令牌时，您会得到另一个与原始令牌无法区分的令牌；如果一个被取消，则另一个也会被取消。您可以根据需要制作任意数量的副本，当您对其中一个副本调用取消时，它们都会被取消。

以下是在多个任务中使用 `CancellationToken` 的步骤：

1. 首先，创建一个新的CancellationToken 。
2. 然后，通过在原始令牌上调用 `clone` 方法来创建原始 `CancellationToken` 的副本。这将创建一个可以由另一个任务使用的新令牌。
3. 将原始或复制的令牌传递给应响应取消请求的任务。
4. 当您要优雅地关闭任务时，请调用原始令牌或复制令牌上的 `cancel` 方法。监听原始或复制令牌上取消请求的任何任务都将被告知关闭。

这是展示了上述步骤代码片段：

```rust
// Step 1: Create a new CancellationToken
let token = CancellationToken::new();

// Step 2: Clone the token for use in another task
let cloned_token = token.clone();

// Task 1 - Wait for token cancellation or a long time
let task1_handle = tokio::spawn(async move {
    tokio::select! {
        // Step 3: Using cloned token to listen to cancellation requests
        _ = cloned_token.cancelled() => {
            // The token was cancelled, task can shut down
        }
        _ = tokio::time::sleep(std::time::Duration::from_secs(9999)) => {
            // Long work has completed
        }
    }
});

// Task 2 - Cancel the original token after a small delay
tokio::spawn(async move {
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;

    // Step 4: Cancel the original or cloned token to notify other tasks about shutting down gracefully
    token.cancel();
});

// Wait for tasks to complete
task1_handle.await.unwrap()
```

使用取消令牌，您不必在令牌取消时立即关闭任务。相反，您可以在终止任务之前运行关闭程序，例如将数据刷新到文件或数据库，或者在连接上发送关闭消息。

## Waiting for things to finish shutting down

一旦告诉其他任务要关闭，您将需要等待它们完成。一种简单的方法是使用 [task tracker](https://docs.rs/tokio-util/latest/tokio_util/task/task_tracker)。task tracker 是任务的集合。task tracker 的 [`wait`](https://docs.rs/tokio-util/latest/tokio_util/task/task_tracker/struct.TaskTracker.html#method.wait) 方法为您提供了一个 future，该 future 只有在其所有包含的 future 都解决并且并且 task tracker 已关闭后才会结束。

下面的示例将产生 10 个任务，然后使用 task tracker 等待它们关闭。

```rust
use std::time::Duration;
use tokio::time::sleep;
use tokio_util::task::TaskTracker;

#[tokio::main]
async fn main() {
    let tracker = TaskTracker::new();

    for i in 0..10 {
        tracker.spawn(some_operation(i));
    }

    // Once we spawned everything, we close the tracker.
    tracker.close();

    // Wait for everything to finish.
    tracker.wait().await;

    println!("This is printed after all of the tasks.");
}

async fn some_operation(i: u64) {
    sleep(Duration::from_millis(100 * i)).await;
    println!("Task {} shutting down.", i);
}
```
