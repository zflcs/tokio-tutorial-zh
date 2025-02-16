# New Tokio release, now with filesystem support

虽然比我最初希望的要长一点（一如既往），但 Tokio 的新版本已经发布了。此版本除了其他功能外，还包含一组新的 API，允许从异步上下文执行文件系统操作。

## Filesystem APIs

与文件（和其他文件系统类型）交互需要*阻塞系统调用，我们都知道阻塞和异步不能混合。因此，从历史上看，当人们问“如何读取和写入文件？”时，答案是使用线程池。这个想法是，当必须执行阻塞读取或写入时，它在线程池上完成，这样它就不会阻塞异步 reactor。

需要单独的线程池来执行文件操作需要消息传递。异步任务必须向线程池发送一条消息，要求其从文件中读取数据，线程池执行读取并用结果填充缓冲区。然后线程池将缓冲区发送回异步任务。这不仅增加了发送消息的开销，而且还需要分配缓冲区来来回发送数据。

现在，有了 Tokio 的[新文件系统 API](https://docs.rs/tokio/0.1/tokio/fs/index.html)，不再需要这种消息传递开销。添加了新的 [`File`](https://docs.rs/tokio/0.1/tokio/fs/struct.File.html) 类型。此类型看起来与 `std` 提供的类型非常相似，但它实现了 `AsyncRead` 和 `AsyncWrite`，因此可以直接在 Tokio 运行时上运行的异步任务中安全地使用它。

由于 [`File`](https://docs.rs/tokio/0.1/tokio/fs/struct.File.html) 类型实现了 `AsyncRead` 和 `AsyncWrite`，因此它的使用方式与 Tokio 中的 TCP 套接字的使用方式非常相似。

截至目前，文件系统 API 非常少。需要实现许多其他 API 才能使 Tokio 文件系统 API 与 std 保持一致，但这些都留给读者作为练习提交 PR！

\* 是的，有些操作系统提供了完全异步的文件系统 API，但这些 API 要么不完整，要么不可移植。

## Standard in and out

Tokio 的此版本还包括异步[标准输入](https://docs.rs/tokio/0.1/tokio/io/fn.stdin.html)和[标准输出](https://docs.rs/tokio/0.1/tokio/io/fn.stdout.html) API。

### `blocking`

这些新 API 的实现得益于新的 [blocking](https://docs.rs/tokio-threadpool/0.1/tokio_threadpool/fn.blocking.html) API，该 API 能够将阻塞当前线程的代码段进行标注。这些阻塞段可以包括阻塞系统调用、等待互斥锁或 CPU 繁重计算。

通过通知 Tokio 运行时当前线程将被阻塞，运行时能够将事件循环从当前线程移动到另一个线程，释放当前线程以允许阻塞。

这与使用消息传递在线程池上运行阻塞操作相反。不是将阻塞操作移动到另一个线程，而是移动整个事件循环。

实际上，将事件循环移到另一个线程比移动阻塞操作要便宜得多。这样做只需要几个原子操作。Tokio 运行时还保留一个待机线程池，以便尽快移动事件循环。

这也意味着使用 `blocking` 注解，`tokio-fs` 必须从 Tokio 运行时的上下文中完成，而不是其他感知 future 的外部 executor。

## Current thread runtime

该版本还包含运行时的“[当前线程](https://docs.rs/tokio/0.1/tokio/runtime/current_thread/index.html)”版本（感谢 [kpp](https://github.com/kpp)）。这与现有运行时类似，但在当前线程上运行所有组件。这允许运行未实现 `Send` 的 Future。
