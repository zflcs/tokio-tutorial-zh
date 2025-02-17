# Announcing Tokio-Compat

Tokio 0.2 的[发布](https://tokio.rs/blog/2019-11-tokio-0-2)是众多贡献者辛勤工作的成果，为 Tokio 带来了多项重大改进。使用 `std::future` 和 async/await 使得使用 Tokio 编写异步代码[更加符合人体工程学](https://tokio.rs/blog/2019-11-tokio-0-2#async-await)，而新的调度程序实现使 Tokio 0.2 的线程池[速度提高了 10 倍](https://tokio.rs/blog/2019-10-scheduler)。但是，更新现有的 Tokio 0.1 项目以使用 0.2 和 `std::future` 带来了一些新的挑战。因此，我们非常高兴地宣布发布 tokio-compat 包来帮助缓解这种转变，通过提供与 Tokio 0.1 和 Tokio 0.2 future 兼容的运行时。

您可以在 [crates.io](https://crates.io/crates/tokio-compat) 和 [GitHub](https://github.com/tokio-rs/tokio-compat) 上找到 `tokio-compat`。

## Motivation

从 `futures` 0.1 到 `std::future` 的转变以及 Tokio 0.2 中 API 的重大变化都使得更新现有项目以从 Tokio 0.2 提供的各种改进中受益变得具有[挑战性](https://users.rust-lang.org/t/failed-to-port-mononoke-to-tokio-0-2-experience-report/32478)。此外，这些重大变化使项目难以逐步迁移：相反，他们必须一次性完成迁移，这需要付出很多努力，有时还会暂停其他工作，直到迁移完成。为了实现增量迁移，我们需要一个兼容层，允许我们在同一个项目中使用针对旧版 Tokio 0.1/`futures` 0.1 API 和新版 Tokio 0.2/`std::future` API 编写的代码。

`futures` crate 的 [`compat` 模块](https://docs.rs/futures/0.3.1/futures/compat/index.html)提供了 `futures` 0.1 和` std::future` future 类型之间的互操作性，例如为实现 `futures` 0.1 `Future` 的类型实现 `std::future::Future`。这是兼容性故事的基础部分，但就其本身而言，它不足以让大多数项目实现逐步更新。大多数使用 Tokio 的代码都依赖于 Tokio 提供的运行时服务。这些运行时服务包括生成其他任务的能力；I/O 驱动程序（允许任务通过操作系统的异步 I/O API 和计时器进行通知）。依赖于 Tokio 0.1 运行时服务的 Future 在尝试访问 Tokio 0.2 运行时上的这些运行时服务（例如通过生成任务或创建计时器）时会发生恐慌，即使它们已转换为 `std::future::Future` trait。这是因为新的运行时没有以与 Tokio 0.1 的 API 兼容的方式提供这些服务。

`tokio-compat` crate 通过提供兼容性运行时来帮助弥补这一差距，该运行时提供与 Tokio 0.1 和 Tokio 0.2 兼容的运行时服务。例如，使用 `tokio-compat`，我们可以编写如下代码：

```rust
use futures_01::future::lazy;

tokio_compat::run(lazy(|| {
    // spawn a `futures` 0.1 future using the `spawn` function from the
    // `tokio` 0.1 crate:
    tokio_01::spawn(lazy(|| {
        println!("hello from tokio 0.1!");
        Ok(())
    }));

    // spawn an `async` block future on the same runtime using `tokio`
    // 0.2's `spawn`:
    tokio_02::spawn(async {
        println!("hello from tokio 0.2!");
    });

    Ok(())
}))
```

类似地，我们可以运行依赖于 0.1 和 0.2 版本运行时服务（如计时器和 I/O 驱动程序）的任务：

```rust
use std::time::{Duration, Instant};
use tokio_compat::prelude::*;

tokio_compat::run_std(async {
    // Wait for a `tokio` 0.1 `Delay`...
    let when = Instant::now() + Duration::from_millis(10);
    tokio_01::timer::Delay::new(when)
        // convert the delay future into a `std::future` that we can `await`.
        .compat()
        .await
        .expect("tokio 0.1 timer should work!");
    println!("10 ms have elapsed");

    // Wait for a `tokio` 0.2 `Delay`...
    tokio_02::time::delay_for(Duration::from_millis(20)).await;
    println!("20 ms have elapsed");
});
```

## Using `tokio-compat`

tokio-compat 的主要用例包括：

- **Incrementally migrating applications**：更新大型项目以使用新的 API 具有挑战性。当可以逐步地、逐个模块地进行这种更改时，或者通过要求添加到项目的新代码使用新的 API 并在现有代码由于其他原因而发生变化时慢慢重写现有代码时，这种更改通常会容易得多。然而，由于 0.1 和 0.2 运行时之间不兼容，对于大多数项目来说这实际上是不可能的。相反，有必要一次性更新所有内容，这需要进行一次大的改变，而这往往会阻碍其他工作。`tokio-compat `可以允许项目逐步迁移。
- **Using legacy libraries in new code**：如果使用 Tokio 0.2 的新项目需要针对 0.1 编写的库的功能，那么它的作者将面临多种选择，但没有一种是特别好的。他们可以在重写依赖项以使用 0.2 版本时阻止其功能的进度，这可能需要很长时间；他们可以重写他们的项目以使用 0.1，放弃 async/await 的所有优点，并且可能需要在依赖项更新时再次重写；或者他们可以自己承担更新依赖项的责任。但是，使用 `tokio-compat`，可以在与 Tokio 0.2 future 相同的运行时上使用 Tokio 0.1 的库中的 future。

在所有这些情况下，`tokio-compat` 希望只是暂时的必需品：理想情况下，大多数代码应该过渡到使用 async/await、`std::future` 和 Tokio 0.2。尽管我们已经尽力使兼容层尽可能轻量，但从定义上讲，它是系统复杂性的一个额外来源。此外，async/await 具有显著的人体工程学优势，使用它的代码更容易理解和修改，因此大多数项目都将从迁移到它中受益匪浅。`tokio-compat` 的作用是使这种转变更容易、更渐进。

### Getting Started

`tokio-compat` 提供的 API 旨在替代 `tokio` 0.1 的 API。`tokio-compat` 提供的[运行时](https://docs.rs/tokio-compat/0.1.0/tokio_compat/runtime/index.html)公开了与 Tokio 0.1 运行时具有相同名称和签名的函数。因此，在很多情况下，开始使用 `tokio-compat` 很简单，添加：

```rust
tokio-compat = { version = "0.1", features = ["rt-full"] }
```

到你的 Cargo.toml，并将引用 `tokio` 0.1 的 `Runtime` 模块的导入和路径更改为 `tokio_compat` 的。例如，

```rust
tokio::runtime::run(future);
```

变为：

```rust
tokio_compat::runtime::run(future);
```

相似地，

```rust
use tokio::runtime::Runtime;

let mut rt = Runtime::new().unwrap();

rt.spawn(future);

rt.shutdown_on_idle()
    .wait()
    .unwrap();
```

变为：

```rust
use tokio_compat::runtime::Runtime;

let mut rt = Runtime::new().unwrap();

rt.spawn(future);

rt.shutdown_on_idle()
    .wait()
    .unwrap();
```

只有 `tokio::runtime` 模块和 `tokio::run` 函数需要替换为 `tokio-compat` 的版本。Tokio 0.1 公开的其他 API，例如 `tokio::net`，将在兼容运行时正常工作（我们将很快讨论一些例外情况）。在兼容运行时运行时，通过 tokio 0.1 的 `spawn` 函数生成 `futures` 0.1 任务的代码将起作用，通过 `tokio` 0.2 的 `spawn` 生成 `std​​::future` 任务的代码也将起作用。

此外，`tokio-compat` 运行时、`TaskExecutor` 和其他类型也提供 `std::future`-compatible 方法。例如，[`Runtime`](https://docs.rs/tokio-compat/0.1.0/tokio_compat/runtime/struct.Runtime.html) 既有 `spawn`，它生成 0.1 futures，又有 `spawn_std`，它生成 `std​​::future` future（或 `async` 函数/块）。有关 spawn 的详细信息，请参阅 `tokio-compat` API 文档中的[此部分](https://docs.rs/tokio-compat/0.1.0/tokio_compat/runtime/index.html#spawning)。

一旦项目在兼容运行时上运行，就很容易逐步迁移到 `std::future` 和 async/await。一种选择是简单地对现有代码库进行“一对一”的翻译：例如，实现 `futures::future::Future` 的类型被重写为实现 `std::future::Future`，使用 future 组合器的代码被更改为使用这些组合器的 `futures` 0.3 版本，等等。在许多情况下，所需的更改相当机械，例如更改导入、将 `Async::NotReady` 重命名为 `Poll::Pending` 等等。但是，`std::future::Future` 的 `poll` 方法的 `Pin<&mut Self>` 接收器类型可能会使迁移手动实现 `Future` 变得具有挑战性。在处理堆栈固定要求时，[`pin-project`](https://crates.io/crates/pin-project) 等 crate 会很有帮助。

然而，在大多数情况下，使用 async/await 语法重写现有代码通常要比手动实现 `Future` 或使用 future 组合器容易得多。尽管从修改的代码行数来衡量，这种变化更大，但 async/await 语法的人体工程学优势可以使迁移变得更加容易——通常，只需简单地删除大量样板代码即可。此外，这样做的好处是可以生成更符合语言习惯、更易读、更易于维护的代码。对于大多数项目来说，改用 async/await 语法是推荐的迁移路径。

### Notes

在当前的 `tokio-compat` v0.1 中，需要注意一些“陷阱”。特别是，需要注意的是，兼容线程池运行时目前不支持 `tokio` 0.1 [`tokio_threadpool::blocking`](https://docs.rs/tokio-threadpool/0.1.16/tokio_threadpool/fn.blocking.html) API。目前，在兼容运行时对旧版本的阻塞进行的调用将会失败。将来，`tokio-compat` 将允许使用 `tokio` 0.2 阻塞 API 透明地替换旧式 `blocking`，但与此同时，需要将此代码转换为调用 `tokio` 0.2 [`task::block_in_place`](https://docs.rs/tokio/0.2.4/tokio/task/fn.block_in_place.html) 和 [`task::spawn_blocking`](https://docs.rs/tokio/0.2.4/tokio/task/fn.spawn_blocking.html) API。由于 [`tokio::fs`](https://docs.rs/tokio/0.1.22/tokio/fs/index.html) 依赖于阻塞 API，因此 Tokio 0.1 版本的 `tokio::fs` 目前也无法在兼容运行时上运行。详情请参阅[此处](https://docs.rs/tokio-compat/0.1.0/tokio_compat/runtime/index.html#blocking)。

此外，需要记住的是，Tokio 0.1 和 Tokio 0.2 在生成任务和关闭运行时方面提供了细微的差别。具体来说，Tokio 0.1 跟踪运行时是否处于空闲状态（即，它没有运行未来），并提供了一个 `Runtime::shutdown_on_idle` 方法，当运行时变为空闲时关闭它。另一方面，Tokio 0.2 的 `spawn` 函数返回 `JoinHandle`，可用于等待生成的任务的完成，而用户则需要等待这些 `JoinHandle` 来确定运行时何时关闭。因此，`tokio-compat` 提供了这两种 API，但需要注意的是，只有在没有 `JoinHandle` 的情况下生成的任务才会“计入”运行时的空闲状态。有关详细信息，请参阅文档中的[此部分](https://docs.rs/tokio-compat/0.1.0/tokio_compat/runtime/index.html#shutting-down)。

## Case Study: Vector

[Vector](https://vector.dev/) 是一款高性能可观察性数据路由器，用于收集和转换日志、指标和事件。Vector 是一款依赖于 Tokio 运行时的生产应用程序，目前依赖于 Tokio 0.1。

将 Vector 从 Tokio 0.1 切换到 Tokio 0.2 后，其维护人员观察到基准测试中性能显著改善：

| benchmark group             |    time (0.1) | thoughput (0.1) |    time (0.2) | throughput (0.2) | speedup |
| --------------------------- | ------------: | --------------: | ------------: | ---------------: | ------: |
| batch 10mb with 2mb batches |    5.5±0.14ms |     1732.8 MB/s |    5.1±0.15ms |      1864.0 MB/s |  x 1.08 |
| buffers/in-memory           |  199.3±2.35ms |       47.8 MB/s |   99.6±4.40ms |        95.7 MB/s |  x 2.00 |
| http/http_gzip              |  191.1±3.17ms |       49.9 MB/s |   94.3±6.24ms |       101.1 MB/s |  x 2.03 |
| http/http_no_compression    |  191.8±4.68ms |       49.7 MB/s |   91.2±2.56ms |       104.6 MB/s |  x 2.10 |
| interconnected              |  238.7±5.66ms |       79.9 MB/s |  209.7±5.32ms |        91.0 MB/s |  x 1.14 |
| pipe                        |  198.4±0.60ms |       48.1 MB/s |  106.0±6.52ms |        90.0 MB/s |  x 1.87 |
| pipe_with_many_writers      | 131.7±12.56ms |       72.4 MB/s | 150.7±10.56ms |        63.3 MB/s |  x 0.86 |
| pipe_with_tiny_lines        |  165.7±0.77ms |      589.2 KB/s |   84.1±0.66ms |      1161.4 KB/s |  x 1.97 |
| transforms                  |  275.7±3.28ms |       38.0 MB/s |  167.5±8.07ms |        62.6 MB/s |  x 1.65 |

请注意，在许多基准测试中，只需更改所使用的 Tokio 运行时，Tokio 0.2 版本（使用 `tokio-compat`）就比 Tokio 0.1 版本快两倍。

更新 Vector 以使用兼容性运行时所需的更改范围也非常小。以下是完整差异：

```shell
diff --git a/src/runtime.rs b/src/runtime.rs
index 07005a7d..4d64a106 100644
--- a/src/runtime.rs
+++ b/src/runtime.rs
@@ -1,15 +1,16 @@
 use futures::future::{ExecuteError, Executor, Future};
+use futures_util::future::{FutureExt, TryFutureExt};
 use std::io;
-use tokio::runtime::Builder;
+use tokio_compat::runtime::Builder;

 pub struct Runtime {
-    rt: tokio::runtime::Runtime,
+    rt: tokio_compat::runtime::Runtime,
 }

 impl Runtime {
     pub fn new() -> io::Result<Self> {
         Ok(Runtime {
-            rt: tokio::runtime::Runtime::new()?,
+            rt: tokio_compat::runtime::Runtime::new()?,
         })
     }

@@ -47,17 +48,17 @@ impl Runtime {
     }

     pub fn shutdown_on_idle(self) -> impl Future<Item = (), Error = ()> {
-        self.rt.shutdown_on_idle()
+        self.rt.shutdown_on_idle().unit_error().boxed().compat()
     }

     pub fn shutdown_now(self) -> impl Future<Item = (), Error = ()> {
-        self.rt.shutdown_now()
+        self.rt.shutdown_now().unit_error().boxed().compat()
     }
 }

 #[derive(Clone, Debug)]
 pub struct TaskExecutor {
-    inner: tokio::runtime::TaskExecutor,
+    inner: tokio_compat::runtime::TaskExecutor,
 }

 impl TaskExecutor {
@@ -71,6 +72,7 @@ where
     F: Future<Item = (), Error = ()> + Send + 'static,
 {
     fn execute(&self, future: F) -> Result<(), ExecuteError<F>> {
-        self.inner.execute(future)
+        self.inner.spawn(future);
+        Ok(())
     }
 }
diff --git a/src/sinks/kafka.rs b/src/sinks/kafka.rs
index 1b99328b..880374b2 100644
--- a/src/sinks/kafka.rs
+++ b/src/sinks/kafka.rs
@@ -6,7 +6,7 @@ use crate::{
     topology::config::{DataType, SinkConfig, SinkContext, SinkDescription},
 };
 use futures::{
-    future::{self, poll_fn, IntoFuture},
+    future::{self, IntoFuture},
     stream::FuturesUnordered,
     Async, AsyncSink, Future, Poll, Sink, StartSend, Stream,
 };
@@ -213,18 +213,14 @@ impl Sink for KafkaSink {
 fn healthcheck(config: KafkaSinkConfig) -> super::Healthcheck {
     let consumer: BaseConsumer = config.to_rdkafka().unwrap().create().unwrap();

-    let check = poll_fn(move || {
-        tokio_threadpool::blocking(|| {
-            consumer
-                .fetch_metadata(Some(&config.topic), Duration::from_secs(3))
-                .map(|_| ())
-                .map_err(|err| err.into())
-        })
-    })
-    .map_err(|err| err.into())
-    .and_then(|result| result.into_future());
+    let task = tokio02::task::block_in_place(|| {
+        consumer
+            .fetch_metadata(Some(&config.topic), Duration::from_secs(3))
+            .map(|_| ())
+            .map_err(|err| err.into())
+    });

-    Box::new(check)
+    Box::new(task.into_future())
 }
```

（感谢 [@LucioFranco](https://github.com/LucioFranco) 在 Vector 中尝试 `tokio-compat`！）

## Conclusion

自 `tokio-core` 时代以来，许多主要的开源项目一直在使用 Tokio。从那时起，Rust 中的异步编程已经取得了令人难以置信的进步，我们希望 `tokio-compat` 能让所有 Tokio 用户尽可能轻松地开始从新的 `std::future`/Tokio 0.2 生态系统中获益。当然，与所有 0.1 版本一样，还有更多工作要做，包括：

- 无缝支持 `tokio-threadpool` 0.1 的[`blocking`](https://docs.rs/tokio-threadpool/0.1.17/tokio_threadpool/fn.blocking.html) API
- `tokio-io` 0.1 的 [`AsyncRead` 和 `AsyncWrite` trait](https://docs.rs/tokio-io/0.1.12/tokio_io/#traits) 的兼容层
- 重新实现 Tokio 0.1 中在 0.2 版本中删除的 API
- 强制性的错误修复和性能改进

您可以在 [GitHub](https://github.com/tokio-rs/tokio-compat) 上找到 `tokio-compat` 存储库。与往常一样，我们非常乐意收到来自 Tokio 社区的反馈和贡献。如果您需要帮助，或者想讨论 `tokio-compat` 的详细信息，请[加入我们的 Discord](https://discord.gg/6yGkFeN) — 欢迎所有人！
