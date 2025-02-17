# Announcing Tokio 1.0

我们很高兴地宣布 Tokio [1.0 版本发布](https://github.com/tokio-rs/tokio/releases/tag/tokio-1.0.0)，它是 Rust 编程语言的异步运行时。Tokio 提供了编写可靠网络应用程序所需的构建块，同时又不影响速度。它带有用于 TCP、UDP、计时器、[多线程](https://tokio.rs/blog/2019-10-scheduler)、[工作窃取调度程序](https://tokio.rs/blog/2019-10-scheduler)等的异步 API。

多年来，我们很高兴看到我们的用户创造了令人惊叹的东西。例如，[Discord](https://blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f) 使用 Tokio 将尾部延迟减少了 5 倍。[Fly.io](https://fly.io/) 发现，借助 Tokio，他们可以轻松满足其性能要求并专注于为客户提供功能。对于 [Zcash 基金会](https://www.zfnd.org/blog/futures-batch-verification/)来说，基于 Tokio 构建使他们能够设计出抗滥用的 API。对于 AWS，他们的 [Lambda](https://aws.amazon.com/lambda/) 团队使用 Tokio 实现了更可靠、更灵活的服务。

本[教程](https://tokio.rs/tokio/tutorial)是学习如何开始使用 Tokio 的最佳场所。

## A stability guarantee

我们四年前首次[发布](https://medium.com/@carllerche/announcing-tokio-df6bb4ddb34) Tokio。从那时起，它已经发生了重大变化。最显著的变化发生在一年前，当时 Rust 添加了 [async 和 await](https://tokio.rs/blog/2019-11-tokio-0-2)。如今，Tokio 更易于使用，功能更强大。这种演变也带来了一些摩擦。它需要库来跟踪这些变化，并且当意外依赖多个版本的 Tokio 时可能会导致令人困惑的错误消息。

Tokio 1.0 的发布结束了这一混乱局面。作为发布的一部分，我们致力于为生态系统提供稳定的基础。我们目前没有 Tokio 2.0 的计划，我们承诺至少在 3 年内不会发布 Tokio 2.0。我们计划维护 Tokio 1.0 分支至少 5 年。Tokio 将保持 6 个月的滚动 MSRV（最低支持 Rust 版本）政策。增加 MSRV 时，新的 Rust 版本必须至少在六个月前发布。

这种稳定并不意味着 Tokio 会停滞不前。我们还有很多工作要做。

## 2021

我们预计来年将重点关注几个领域：`Stream`、`io_uring`、`tracing` 集成和 `Tokio` 栈。

### `Stream`

目前，[`tokio-stream`](https://docs.rs/tokio-stream) crate 提供了基于 Stream trait 的异步迭代工具。将 [`Stream` trait](https://docs.rs/futures-core/0.3/futures_core/stream/trait.Stream.html) 从 [`futures-core`](https://docs.rs/futures-core/) 移至 Rust 标准库的 [RFC](https://github.com/rust-lang/rfcs/pull/2996) 正在等待批准。

一旦标准库提供了 Stream trait，Tokio 的 stream 工具将会移入 Tokio crate 本身。

### `io_uring`

`io_uring` 是一个新的 Linux 接口，为所有类型的 I/O（包括磁盘）提供异步操作，同时减少所需的系统调用数量。目前，在 Linux 上，Tokio 使用 `epoll(7)` 接口，众所周知，该接口不适用于磁盘访问。为了解决这个问题，Tokio 通过将同步操作分派到线程池来提供异步文件系统 API。虽然这种方法可行，但并不是最佳选择。

通过利用 `io_uring`，Tokio 将能够提供真正的异步文件系统操作。此外，`io_uring` 的早期基准测试非常有希望，我们希望这些性能改进能够延续到 Tokio。

### `tracing`

围绕运营 Tokio 应用程序定义一流的故事将成为 2021 年的重点。这个故事的很大一部分将是围绕 tracing 和 metrics 的工具。[`tracing`](https://crates.io/crates/tracing) crate 已经提供了大部分所需的基础设施。它实现了用于结构化诊断的外观层和数据模型，允许库和应用程序发出可按多种方式使用的机器可读数据。

在接下来的一年里，我们将在 tracing 和 Tokio 栈的其余部分之间建立更深层次的集成。这种集成将有助于提供对 Tokio 内部的必要可见性，以解答问题。问题包括如何调整 Tokio 运行时以减少尾部延迟、了解当前正在运行的任务或哪些锁的竞争最激烈。

此外，我们将继续改善和发展其余的 `tracing` 生态系统。这包括针对核心 `tracing` 和 `tracing-core` crate 的新版本的工作，为 Tokio 栈中的库添加更丰富的工具，并提供与其他诊断和可观察性工具的集成，例如 [`tracing-opentelemetry`](https://crates.io/crates/tracing-opentelemetry)。

### The Tokio stack

Tokio 为标准原语（如套接字和计时器）提供了运行时和异步 API，但网络应用程序通常会使用更高级别的协议，如 HTTP 和 gRPC。Tokio 栈包括用于 HTTP 的 Hyper 和用于 gRPC 的 Tonic，以满足这些需求。今天，我们发布了支持 Tokio 1.0 的 [Hyper 0.14](https://seanmonstar.com/post/638320652536922112/hyper-v014)。Tonic 将很快更新。

随着 Tokio 1.0 的推出，我们还将重点关注 [Tower](https://crates.io/crates/tower)，这是一组用于构建可靠客户端和服务器的可重复使用组件。

## Thanks

如果没有众多贡献者的支持，Tokio 就不可能实现，尤其是 [Alice Ryhl](https://github.com/darksonn)，她对这个项目做出了巨大贡献。感谢 Tokio 栈库的领导者：[Sean McArthur](https://github.com/seanmonstar/) (Hyper)、[Lucio Franco](https://github.com/luciofranco) (Tonic)、[Eliza Weisman](https://github.com/hawkw) (Tracing) 和 [Thomas De Zeeuw](https://github.com/Thomasdezeeuw)(Mio)。此外，如果没有 [Aaron Turon](https://github.com/aturon) 和 [Alex Crichton](https://github.com/alexcrichton) 的早期参与，Tokio 也不可能成功。

最后，我们要向那些为 Tokio 提供资金支持的公司致以衷心的感谢：[Mozilla](https://mozilla.org/)、[Dropbox](https://dropbox.com/)、[Buoyant](https://buoyant.io/) 和 [AWS](https://aws.amazon.com/)。
