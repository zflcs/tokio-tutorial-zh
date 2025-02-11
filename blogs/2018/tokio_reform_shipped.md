# Tokio Reform is Shipped and the Road to 0.2

大家好！

我很高兴地宣布，今天，[reform RFC](https://github.com/tokio-rs/tokio-rfcs/blob/master/text/0001-tokio-reform.md) 中提出的变更已作为 `tokio` 0.1 发布到 [crates.io](https://crates.io/crates/tokio)。

主要变化如下：

- 添加默认的全局事件循环，绝大多数情况下无需设置和管理自己的事件循环。
- 将所有任务执行功能与 Tokio 分离。

## The new global event loop

到目前为止，创建事件循环都是一个手动过程。尽管绝大多数 Tokio 用户都会设置 reactor 来做同样的事情，但每个人每次都必须这样做。部分原因是在 Tokio reactor 的线程上运行代码与在另一个线程（如线程池）上运行代码之间存在显著差异。

Tokio 改革的关键见解是 Tokio reactor 实际上不必成为 Executor。换句话说，在这些变化之前，Tokio reactor 既能为 I/O 资源提供动力，又能管理执行用户提交的任务。

现在，Tokio 提供了一个 reactor 来独立于任务 executor 驱动 I/O 资源（如 `TcpStream` 和 `UdpSocket`）。这意味着可以从任何线程轻松创建 Tokio 支持的网络类型，从而轻松创建单线程或多线程 Tokio 支持的应用程序。

对于任务执行，Tokio 提供了 [`current_thread`](https://docs.rs/tokio-current-thread) 执行器，其行为与 tokio-core 内置的 executor 类似。计划最终将此执行器移入 [`futures`](https://github.com/rust-lang-nursery/futures-rs) crate 中，但目前它由 Tokio 直接提供。

## The road to 0.2

Tokio 改革变更已发布为 0.1。依赖项（[`tokio-io`](https://github.com/tokio-rs/tokio-io)、[`futures`](https://github.com/rust-lang-nursery/futures-rs)、[`mio`](https://github.com/carllerche/mio) 等）的版本尚未增加。这使得 `tokio` crate 的发布对生态系统的干扰最小。

计划是让此版本中所做的更改得到一些使用，然后再提交。任何需要重大更改的修复都可以与所有其他包的发布同时完成。目标是在 6-8 周内实现这一目标。因此，请尝试今天发布的更改并提供反馈。

## Rapid iteration

这仅仅是个开始。Tokio 有雄心勃勃的目标，即提供额外的功能，以获得在 Rust 中构建异步 I/O 应用程序的出色“开箱即用”体验。

为了尽快实现这些目标，同时又不造成不必要的生态系统破坏，我们将采取一些措施。

首先，与 [`futures` 0.2 版本](https://tokio.rs/blog/2018-02-tokio-reform-shipped#)类似，`tokio` crate 将转变为更像一个门面。traits 和类型将被分解为多个子 crate，并由 `tokio` 重新导出。应用程序作者将能够直接依赖 `tokio`，而库作者将挑选他们希望用作其库的一部分的特定 Tokio 组件。

每个子 crate 都会明确标明其稳定性级别。显然，futures 0.2 版本即将带来重大变化，但此后，基本构建块将力求保持至少一年的稳定性。更多实验性的 crate 将保留以更快的速度发布重大变化的权利。

这意味着 `tokio` crate 本身将能够以更快的速度迭代，同时库生态系统保持稳定。

0.2 之前的版本也将是一个实验阶段。将以实验的形式向 Tokio 添加附加功能。在 0.2 版本发布之前，我们将发布一个 RFC，涵盖我们希望在该版本中包含的功能。

## Open question

剩下的一个问题是如何处理 `tokio-proto`。它是作为初始 Tokio 版本的一部分发布的。从那时起，焦点已经转移，并且该 crate 没有受到足够的关注。

我在[这里](https://github.com/tokio-rs/tokio/issues/118)发布了一个问题来讨论如何处理该 crate。

## Looking Forward

请试用今天发布的更改。同样，接下来的几个月是我们发布下一个版本之前的试验期。所以，现在是尝试并提供反馈的时候了。

在此期间，我们将整合这项工作以在 [Tower](https://github.com/tower-rs/tower) 中构建更高级别的原语，这是由 [Conduit](https://github.com/runconduit/conduit) 项目的生产运营需求推动的。
