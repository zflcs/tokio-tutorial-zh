# Announcing Tokio 0.3 and the path to 1.0

Tokio 团队很高兴地宣布 Tokio 0.3 的发布。此版本可用作 Tokio 1.0 测试版。API 的缺陷已修复。此版本是验证更改并在将其作为 1.0 版本的一部分进行稳定化之前进行验证的机会。由于大多数问题都很小，我们预计从 0.2 升级到 0.3 将很容易。

## Plan for 1.0

Tokio 团队计划在 2020 年 12 月底之前发布 Tokio 1.0。这个日期很快就要到了。我们希望您（Tokio 社区）试用 Tokio 0.3 并通过 [GitHub Issues](https://github.com/tokio-rs/tokio/issues) 或我们的 [Discord 频道](https://discord.gg/tokio)向我们提供反馈。一旦我们发布 Tokio 1.0，我们将承诺以下稳定性保证：

- 至少5年的维护。
- 距离假设的 2.0 版本发布至少还有 3 年时间。

## What’s new?

Tokio 0.3 的主要变化是：

- IO trait 的改变。
- 新的运行时构建器。
- I/O 驱动程序经过彻底改造。
- 该 API 具有面向 future 的特性。

您可以在[变更日志](https://github.com/tokio-rs/tokio/releases/tag/tokio-0.3.0)中找到完整列表。

## Changes to IO traits

按照@sfackler 的 [RFC](https://github.com/rust-lang/rfcs/pull/2930) 的建议，我们改变了 `AsyncRead` 和 `AsyncWrite` trait 以支持读取未初始化的内存。此变更仅影响实现 `AsyncRead` 或 `AsyncWrite` 特征或手动调用 `poll_*` 方法的用户。如果使用 `AsyncReadExt` 或 `AsyncWriteExt` trait，则无需进行任何变更。

下面是新的 AsyncRead 的简化版本：

```rust
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>, 
        buf: &mut ReadBuf<'_>
    ) -> Poll<Result<()>>;
}

pub struct ReadBuf<'a> {
    buf: &'a mut [MaybeUninit<u8>],
    filled: usize,
    initialized: usize,
}

impl<'a> ReadBuf<'a> {
    // functions here, see RFC
}
```

Tokio 0.2 的 `poll_read` 方法的 `&mut [u8]` 参数有一些意想不到的尖锐边缘：`AsyncRead` trait 的实现者（而不是最终消费者）可以读取存储在 `&mut [u8]` 切片中的数据。如果程序员创建一个可变的字节切片（`&mut [u8]`），引用未初始化的内存（例如，向量中的多余容量），那么读取 &mut [u8] 数据将导致未定义的行为。新的设计通过提供跟踪未初始化内存的 `ReadBuf` 结构来缓解这个问题，从而无需初始化内存。

此外，`poll_read_buf` 和 `poll_write_buf` 方法已从这两个特征中删除。实际上，这些方法很少实现。向量操作将直接在支持它们的类型上实现。

## Feature flag simplification

我们减少了使用 Tokio 核心功能所需的功能标记数量。`dns`、`tcp`、`udp` 和 `uds` 功能标记被合并为单个 `net` 功能标记。我们还将 `rt-core` 和 `rt-util` 合并为单个 `rt` 功能标志。这个 `rt` 功能标志现在包含执行 future 所需的一切，除了多线程运行时，它现在位于 `rt-multi-thread` 功能标志下（在 0.2 中称为 `rt-threaded`）。

- `dns`, `tcp`, `udp` 和 `uds` -> `net`
- `rt-util` 和 `rt-core` -> `rt`
- `rt-threaded` -> `rt-multi-thread`

## Runtime and Builder refactor

Tokio 团队改进了运行时 `Builder`，以在功能标志排列之间提供更好的一致性，并提高抗误用能力。为了实现这一点，我们向 `Builder` 类型添加了两个新的构造函数，以支持构建 Tokio 运行时的两种变体。此外，使用 `Runtime::new` 时，Tokio 将不再根据功能标志选择运行时变体，而是始终使用多线程运行时，该运行时仅在 `rt-multi-thread` 功能标志下可用。此外，我们将 `core_threads` 构建器方法重命名为更准确的 `worker_threads`。

我们还重构了运行时模块。Tokio 0.2 公开了两种与 Tokio `运行时交互的类型：Runtime` 和 `Handle`。`Runtime` 有一个需要 `&mut self` 的 `block_on` 方法，而 `Handle` 有一个需要 `&self` 的 `block_on` 方法。在 Tokio 0.3 中，我们将 `Runtime` 和 `Handle` 合并为单个 `Runtime`。新运行时上的 `block_on` 方法采用 `&self`，这意味着它可以毫无问题地放入 `Arc` 中。

配置具有 6 个工作线程的多线程运行时的示例：

```rust
use tokio::runtime::Builder;

let rt = Builder::new_multi_thread()
    .worker_threads(6)
    .enable_all()
    .build()
    .unwrap();

rt.block_on(async {
    println!("Hello world!");
});
```

## Methods changed to `&self`

我们已经彻底改进了在套接字类型中处理 waker 的方式，以允许通过 `&self` 而不是 `&mut self` 进行并发调用。为了实现这一点，我们必须重写内部处理 waker 的方式。问题在于基于 `poll_*` 的方法的工作方式。根据 future，它规定 `poll_*` fn 应该只存储调用它的最后一个唤醒程序。

> 请注意，在多次调用 poll 时，只有传递给最近一次调用的 Context 中的 Waker 才应被安排接收唤醒。

来自 [`Future`](https://doc.rust-lang.org/std/future/trait.Future.html)

这意味着这些 `poll_*` 方法不能同时调用或唤醒。为了使 `&self` 适用于我们的套接字类型的 `async fn`，我们重构了 io 驱动程序以使用内部侵入式链接列表来存储 waker。通过这样做，我们的套接字类型在调用时可以接受 `&self` 而不是 `&mut self`。

## Removal of non-1.0 crates and future proofing for 1.0

在推进 1.0 的过程中，我们从公共 API 中删除了许多非 1.0 依赖项。其中包括 `bytes` 和 `mio`。这将使我们能够在继续创新底层包的同时，推动面向 future 且更稳定的 1.0 版本。

## Mio 0.7

我们最终还将 `mio` 升级到了 0.7，它最初于 2019 年 12 月以 alpha 版本发布，但未能进入 Tokio 0.2。从那时起，`mio` 0.7 逐渐成熟，并最终进入 Tokio 0.3 的这个版本。这将一些关键依赖项（如 `winapi`）从 0.2 升级到 0.3。此外，`mio` 0.7 现在附带了 [`wepoll`](https://github.com/piscisaureus/wepoll) 的纯 rust 实现，这是一个为基于 Windows 的应用程序实现 `epoll` API 的库。

## Compatibility between 0.2 and 0.3

可以通过 [`tokio-compat-02`](https://docs.rs/tokio-compat-02) crate 逐步升级到 Tokio 0.3。此 crate 将从 Tokio 0.2 启动单线程后台运行时，并允许您在其上下文中包装任何 future。这样您就可以在任何运行时内运行需要 Tokio 0.2 的库，包括 Tokio 0.3。以下是如何使用需要 Tokio 0.2 的 hyper 0.13 发送请求的示例。

```rust
use hyper::{Client, Uri};
use tokio_compat_02::FutureExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new();

    // This will not panic because we are wrapping it in the
    // Tokio 0.2 context via the `FutureExt::compat` fn.
    client
        .get(Uri::from_static("http://tokio.rs"))
        .compat()
        .await?;

    Ok(())
}
```

## Conclusion

自 Tokio 0.2 发布以来，已有 50 多位贡献者为我们提供帮助。我们非常感谢社区帮助我们发现错误并帮助我们修复它们。随着我们继续向 1.0 迈进，反馈将比以往任何时候都更加重要，因此请随时打开 issue 或加入我们的 [Discord](https://discord.gg/tokio) 来帮助我们实现目标。
