# Setup

这个指导将会带你异步异步构建 Redis 的客户端和服务端。我们将从使用 Rust 进行异步编程的基础开始。我们将实现 Redis 命令的一个子集，从而展开对 Tokio 的全面介绍。

## Mini-Redis

这个指导手册构建的项目可以在 [Mini-Redis on GitHub](https://github.com/tokio-rs/mini-redis) 获得。设计 Mini-Redis 的主要目的是用于学习 Tokio，因此它的注释很好，但是这也意味着 Mini-Redis 缺少了在真正的 Redis 库中想要的一些功能。您可以在 [crates.io](https://crates.io/) 上找到可用于生产的 Redis 库。

我们将在指导手册中直接使用 Mini-Redis。这使得我们可以在本教程中使用 Mini-Redis 的部分功能，然后在本教程的后面部分实现它们。

## 获取帮助

无论何时，如果您遇到困难，您都可以在 [Discord](https://discord.gg/tokio) 或 [GitHub 讨论](https://github.com/tokio-rs/tokio/discussions) 中寻求帮助。不用担心问“初学者”的问题。我们都是从一个地方开始的，很乐意提供帮助。

## 前提条件

读者应该熟悉 [Rust](https://rust-lang.org/)，而 [Rust book](https://doc.rust-lang.org/book/) 是很好的学习资源。

尽管不是必要，但使用 [Rust 标准库](https://doc.rust-lang.org/std/) 或者其他语言编写网络代码的经验也很有帮助。

无需预先具备 Redis 知识。

## Rust

在开始之前，你应该确保已经安装了 [Rust 工具链](https://www.rust-lang.org/tools/install) 并准备就绪。如果你还没有，最简单的安装方法是使用 [rustup](https://rustup.rs/)。

本教程要求 Rust 最低版本 `1.45.0`，但建议使用最新稳定版本的 Rust。

要检查您的计算机上是否安装了 Rust，请运行以下命令：

```shell
rustc --version
```

您应该看到类似 `rustc 1.46.0 (04488afe3 2020-08-24)` 的输出。

## Mini-Redis server

接下来，安装 Mini-Redis 服务器。这将用于在构建客户端时对其进行测试。

```shell
cargo install mini-redis
```

通过启动服务器确保已成功安装：

```shell
mini-redis-server
```

然后，在单独的终端窗口中，尝试使用 `mini-redis-cli` 获取 `foo`：

```shell
mini-redis-cli get foo
```

您应该看到 `(nil)`。

## 准备就绪

一切准备就绪。转到下一页编写您的第一个异步 Rust 应用程序。
