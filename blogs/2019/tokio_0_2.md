# Announcing Tokio 0.2 and a Roadmap to 1.0

我们非常高兴地宣布 Tokio 0.2。这是基于 `async` / `await` 和过去三年积累的经验对 Tokio 进行的彻底改造。

将以下内容添加到 `Cargo.toml`：

```rust
tokio = { version = "0.2", features = ["full"] }
```

这是一次重大更新，因此库的几乎所有部分都发生了变化。重点介绍以下几点：

- 基于 `async` / `await`，实现卓越的人体工程学。
- 一个全新的、更快的调度程序。
- 专注于使 Tokio 依赖性尽可能轻量。

## `async` / `await`

如果您迄今为止使用过 Tokio，那么您一定熟悉编写异步 Rust 的“旧”方式……这并不好玩。大多数人最终都手动编写状态机。这很麻烦，而且容易出错。

从 Rust 1.39 开始，`async` / `await` 在稳定频道上可用，Tokio 0.2 充分利用了它。只要有可能，Tokio 就会提供基于 `async fn` 的 API，通常以 `std` 为模型。

例如，从 `TcpListener` 接受套接字是使用异步 `accept` 函数完成的：

```rust
let (mut socket, _peer_addr) = listener.accept().await?;
```

I/O 操作以 `async` 函数的形式提供：

```rust
let n = socket.read(&mut buf).await?;

let n = socket.write(&buf).await?;
```

这同样适用于 Tokio API 层的其余部分。

## Starting the Tokio runtime

现在可以使用过程宏来设置 Tokio 应用程序的入口点：

```rust
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

这将启动 Tokio 运行时和为应用程序提供支持的所有必要基础设施。

## A new scheduler

新的 Tokio 配备了一个从头开始构建的调度程序，以利用新的异步任务系统。它基于从 Tokio 0.1 获得的经验以及在其他生态系统（如 Go、Erlang、Java 等）中投入的所有辛勤工作。

仍有改进空间，但使用 [Hyper](https://hyper.rs/) 进行的初步测试表明，旧调度程序和新调度程序之间的宏观级别基准测试速度提高了 30% 以上。

您可以在[此处](https://tokio.rs/blog/2019-10-scheduler/)阅读更多相关信息。

自从它登陆 master 以来，一些 Tokio 用户一直在尝试使用新的调度程序，并在他们的应用程序中看到了一些非常令人印象深刻的实际改进。希望他们能很快在博客上介绍这些改进！

## A lightweight Tokio dependency

到目前为止，用户对 Tokio 最大的抱怨之一是依赖项的负担。添加对 Tokio 的依赖项过去会增加大量传递依赖项并增加编译时间。

例如，我的笔记本电脑上的 "hello world" Tokio 0.1 应用程序将引入 43 个包并花费 50 秒进行编译（不包括下载依赖项所花费的时间）。

Tokio 从两个方面解决了这个问题。首先，大多数 Tokio 组件已被合并到一个单独的 crate 中：tokio 和传递依赖关系被大幅度精简。其次，Tokio 组件已通过功能标记选择加入，而不是始终启用。简单地引入 tokio 依赖项只能为您带来一些特性。

要开始使用 Tokio 0.2，您需要请求功能标志。完整的功能标志包括所有内容，是一种简单的入门方式：

```rust
tokio = { version = "0.2", features = ["full"] }
```

在我的笔记本电脑上，这会将 crate 的总数减少到 23 个。编译时间仅下降到 40 秒。

当 Tokio 用户开始仅请求运行应用程序所需的组件时，真正的好处就开始显现。要运行 TCP echo 服务器示例，需要 `io-util`、`rt-threaded` 和 `tcp` 功能标志。现在，Tokio 引入了 13 个包，编译需要 13 秒。

在修剪依赖项方面还有更多工作要做。Mio 0.7 可以修剪更多依赖项，但未包含在 Tokio 0.2 中。Tokio 0.3 将包含 Mio 0.7。

## Thanks

当然，如果没有我们出色的团队和贡献者为这个版本做出的努力，这一切都不可能实现。许多人提交了 PR，从文档修复到将整个包迁移到 `std::future::Future`。我想点出一些名字（无特定顺序）：

- [@hawkw](https://github.com/hawkw)
- [@ipetkov](https://github.com/ipetkov)
- [@jonhoo](https://github.com/jonhoo)
- [@LucioFranco](https://github.com/LucioFranco)
- [@seanmonstar](https://github.com/seanmonstar)
- [@taiki-e](https://github.com/taiki-e)

首先感谢所有参与 Mio 0.7 开发的人们，遗憾的是它未能包含在本次发布中，但很快就会发布！

- [@dtacalau](https://github.com/dtacalau)
- [@kleimkuhler](https://github.com/kleimkuhler)
- [@PerfectLaugh](https://github.com/PerfectLaugh)
- [@piscisaureus](https://github.com/piscisaureus)
- [@Thomasdezeeuw](https://github.com/Thomasdezeeuw)

非常感谢 [Linkerd](https://github.com/linkerd/linkerd2)（代理用 Rust 编写）的创造者 [Buoyant](https://buoyant.io/)，他们赞助了大部分工作。

## A Roadmap to 1.0

话虽如此，我们还是发布了 0.2 版。“为什么不发布 1.0 版？”这个问题已经出现过好几次了。毕竟，Tokio 0.1 已经稳定了三年。简短的回答是：因为还不到时候。没有人比我们更愿意发布 Tokio 1.0。这也不是什么急事。

毕竟，`async` / `await` 几周前才进入稳定的 Rust 频道。目前还没有进行过重大的生产验证，除了 fuchsia，这似乎是一个相当特殊的用例。Tokio 的这个版本包含重要的新代码和带有功能标记的新策略。此外，仍有一些悬而未决的大问题，例如对 `AsyncRead` 和 `AsyncWrite` 的[拟议更改](https://github.com/tokio-rs/tokio/pull/1744)。

一旦 API 被证明可以处理实际生产案例，Tokio 1.0 就会发布。

## Tokio 1.0 in Q3 2020 with LTS support

Tokio 1.0 的发布将不迟于 2020 年第三季度。它还将附带“长期支持”保证：

- 至少5年的维护。
- 距离假设的 2.0 版本发布至少还有 3 年时间。

当 Tokio 1.0 于 2020 年第三季度发布时，将保证持续支持、安全修复和关键错误修复至少持续到 2025 年第三季度。Tokio 2.0 至少要到 2023 年第三季度才会发布（不过，理想情况下永远不会有 Tokio 2.0 发布）。

## How to get there

虽然 Tokio 0.1 可能应该是 1.0，但 Tokio 0.2 将是真正的 0.2 版本。每 2 至 3 个月将发布重大变更，直到 1.0 版本。这些变化将比从 0.1 到 0.2 的变化小得多。预计 1.0 版本将与 0.2 版本非常相似。

## What is expected to change

最大的变化将是 `AsyncRead` 和 `AsyncWrite` trait。根据过去 3 年获得的经验，有几个问题需要解决：

- 能够安全地使用未初始化的内存作为读取缓冲区。
- 实用的 read vectored 和 write vectored API。

有几种策略可以解决这些问题。这些策略需要研究，解决方案需要验证。您可以查看[此评论](https://github.com/tokio-rs/tokio/pull/1744#issuecomment-553575438)以了解问题的详细说明。

另一项已筹备一段时间的重大变化是更新 Mio。Mio 0.6 于近 4 年前首次发布，自那时起没有发生过重大变化。Mio 0.7 已经开发了一段时间。它包括对 Windows 支持的完全重写以及改进的 API。不久将对此进行更多介绍。

最后，现在 API 开始稳定，我们将努力编写文档。Tokio 0.2 在更新网站之前发布，许多旧内容将不再相关。在接下来的几周内，预计将在那里看到更新。

所以，我们已做好了充分准备。我们希望您喜欢这个 0.2 版本，并期待您的反馈和帮助。
