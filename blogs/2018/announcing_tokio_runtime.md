# Announcing the Tokio runtime

我很高兴宣布 Tokio 的新版本。此版本包括 Tokio Runtime 的第一个迭代。

这就是基于多线程 Tokio 的服务器的编写方式：

```rust
extern crate tokio;

use tokio::net::TcpListener;
use tokio::prelude::*;

fn process(s: TcpStream)
  -> impl Future<Item = (), Error = ()> + Send
{ ... }

let addr = "127.0.0.1:8080".parse().unwrap();
let listener = TcpListener::bind(&addr).unwrap();

let server = listener.incoming()
    .map_err(|e| println!("error = {:?}", e))
    .for_each(|socket| {
        tokio::spawn(process(socket))
    });

tokio::run(server);
```

其中 `process` 表示一个用户定义的函数，它接受一个套接字并返回一个处理它的 future。对于 echo 服务器，这可能是从套接字读取所有数据并将其写回到同一套接字。

使用运行时的[指南](https://tokio.rs/docs/getting-started/hello-world/)和[示例](https://github.com/tokio-rs/tokio/tree/v0.1.x/tokio/examples)已更新。

## What is the Tokio Runtime?

Rust 异步 stack 正在演变为一组松散耦合的组件。要运行基本的网络应用程序，您至少需要一个异步任务 executor 和一个 Tokio reactor 实例。因为一切都是分离的，所以这些不同的组件有多个选项，但这会给所有应用程序增加一堆样板。

为了缓解这种情况，Tokio 现在提供了运行时的概念。这是运行应用程序所需的所有各种组件的预配置包。

该运行时的初始版本包括 reactor 以及用于调度和执行应用程序代码的基于[工作窃取](https://en.wikipedia.org/wiki/Work_stealing)的线程池。这为应用程序提供了多线程默认设置。

默认的工作窃取对于大多数应用程序来说是理想的。它使用与 Go、Erlang、.NET、Java（ForkJoin 池）等类似的策略…… Tokio 提供的实现是为在单个线程池上多路复用许多**不相关**的任务的用例而设计的。

## Using the Tokio Runtime

如上例所示，使用 Tokio 运行时的最简单方法是使用两个函数：

- `tokio::run`
- `tokio::spawn`

第一个函数接受一个 Future 来为应用程序提供种子并启动运行时。大致来说，它执行以下操作：

1. 启动 reactor。
2. 启动线程池。
3. 将 future spawn 到线程池中。
4. 阻塞线程直到运行时变为空闲。

一旦所有产生的 future 都已完成并且绑定到 reactor 的所有 I/O 资源都被删除，运行时就会变为空闲状态。

在运行时上下文中。应用程序可以使用 `tokio::spawn` 在线程池中生成其他 future。

或者，可以直接使用 [`Runtime`](https://tokio.rs/blog/2018-03-tokio-runtime#) 类型。这为设置和使用运行时提供了更大的灵活性。

## Future improvements

这只是 Tokio 运行时的初始版本。即将发布的版本将包含对基于 Tokio 的应用程序有用的附加功能。即将发布的博客文章将更详细地介绍路线图。

正如之前提到的，我们的目标是尽早并经常发布。提供新功能，让社区能够尝试它们。在接下来的几个月中，整个 Tokio stack 都会发布一个重大版本，因此需要在此之前发现 API 中的任何更改。

## Tokio-core

`tokio-core` 也发布了新版本。此版本更新了 `tokio-core`，以便在底层使用 `tokio`。这使得所有当前依赖于 `tokio-core`（如 Hyper）的现有应用程序和库能够使用 Tokio 运行时带来的改进，而无需进行重大更改。

考虑到未来几个月预计将发生的客户流失量，我们希望帮助缓解版本间的过渡。






