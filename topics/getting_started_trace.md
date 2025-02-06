# Getting started with Tracing

[tracing](https://docs.rs/tracing) crate 是用于检测 Rust 程序以收集结构化的、基于事件的诊断信息的框架。

在像 Tokio 这样的异步系统中，解释传统的日志消息通常很具有挑战性。由于各个任务在同一个线程上多路复用，相关事件和日志行混合在一起，使得难以追踪逻辑流。`tracing` 扩展了日志式诊断，允许库和应用程序记录结构化事件以及有关时间性和因果关系的附加信息。与日志消息不同，`tracing` 中的 [`Span`](https://docs.rs/tracing/latest/tracing/#spans) 具有开始和结束时间，可以通过执行流进入和退出，并且可以存在于类似 Span 的嵌套树中。为了表示在某一时刻发生的事情，`tracing` 提供了事件这一补充概念。[`Span`](https://docs.rs/tracing/latest/tracing/#spans) 和 [`Event`](https://docs.rs/tracing/latest/tracing/#events) 都是结构化的，能够记录类型化数据以及文本消息。

您可以使用 `tracing` 来：

- 向 [`OpenTelemetry`](https://docs.rs/tracing-opentelemetry) 收集器发送分布式跟踪
- 使用 [`Tokio Console`](https://docs.rs/console-subscriber) 调试您的应用程序
- 记录到 [`stdout`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html)、[日志文件](https://docs.rs/tracing-appender/latest/tracing_appender/)或 [`journald`](https://docs.rs/tracing-journald/latest/tracing_journald/)
- [profile](https://docs.rs/tracing-timing/latest/tracing_timing/) 您的应用程序花费的时间

## Setup

首先，添加 [`tracing`](https://docs.rs/tracing) 和 [`tracing-subscriber`](https://docs.rs/tracing-subscriber) 作为依赖：

```Rust
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```

[`tracing`](https://docs.rs/tracing) crate 提供了我们用来发出跟踪的 API。[`tracing-subscriber`](https://docs.rs/tracing-subscriber) crate 提供了一些基本工具，用于将这些跟踪转发到外部侦听器（例如，`stdout`）。

## Subscribing to Traces

如果您正在编写可执行文件（而不是库），则需要注册 tracing [subscriber](https://docs.rs/tracing/latest/tracing/#subscribers)。Subscribers 是处理应用程序及其依赖项发出的跟踪信息的类型，可以执行计算指标、监视错误以及向外界（例如，`journald`，`stdout`，或者 `open-telemetry daemon`）重新发出跟踪信息等任务。

在大多数情况下，您应该尽早在 `main` 函数中注册 subscriber。例如，[`tracing-subscriber`](https://docs.rs/tracing-subscriber) 提供的打印格式化跟踪信息和事件到 `stdout` 的 [`FmtSubscriber`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html) 类型可以这样注册：

```rust
#[tokio::main]
pub async fn main() -> mini_redis::Result<()> {
    // construct a subscriber that prints formatted traces to stdout
    let subscriber = tracing_subscriber::FmtSubscriber::new();
    // use that subscriber to process traces emitted after this point
    tracing::subscriber::set_global_default(subscriber)?;
    ...
}
```

如果您现在运行您的应用程序，您可能会看到一些由 Tokio 发出的跟踪事件，但您需要修改您自己的应用程序以发出跟踪，以最大限度地利用 `tracing`。

### Subscriber Configuration

在上面的示例中，我们已经配置了其默认配置的 [`FmtSubscriber`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html)。但是，`tracing-subscriber` 还提供了多种配置 [`FmtSubscriber`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html) 的行为的方法，例如自定义输出格式，包括日志中的其他信息（例如线程ID或源代码位置），并将日志写入其他地方，而不是 `stdout`。

例如：

```rust
// Start configuring a `fmt` subscriber
let subscriber = tracing_subscriber::fmt()
    // Use a more compact, abbreviated log format
    .compact()
    // Display source code file paths
    .with_file(true)
    // Display source code line numbers
    .with_line_number(true)
    // Display the thread ID an event was recorded on
    .with_thread_ids(true)
    // Don't display the event's target (module path)
    .with_target(false)
    // Build the subscriber
    .finish();
```

有关可用配置选项的详细信息，请参见 [`tracing_subscriber::fmt` 文档](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html#configuration)。

除了 [`tracing-subscriber`](https://docs.rs/tracing-subscriber) 提供的 [`FmtSubscriber`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html) 类型外，其他 `Subscriber` 可以实现自己的记录 `tracing` 数据的方式。包括替代输出格式，分析和聚合，以及与其他系统（例如分布式跟踪或日志聚合服务）集成。许多 crate 提供了可能感兴趣的其他 `Subscriber` 实现。请参阅[此处](https://docs.rs/tracing/latest/tracing/index.html#related-crates)以获取其他 `Subscriber` 实现的（不完整）列表。

最后，在某些情况下，将多种不同的记录跟踪方式组合在一起以构建实现多种行为的单个 `Subscriber` 可能会很有用。为此，`tracing-subscriber` crate 提供了一个 [`Layer`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/index.html) trait，表示可以与其他 `Layers` 组合在一起形成 `Subscriber` 的组件。有关使用 `Layers` 的详细信息，请参阅[此处](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/index.html)。

## Emitting Spans and Events

发出 Span 的最简单方法是使用 `tracing` 提供的 proc-macro 注释[instrument](https://docs.rs/tracing/latest/tracing/attr.instrument.html)，它会重写函数主体以在每次调用时发出 span；例如：

```rust
#[tracing::instrument]
fn trace_me(a: u32, b: u32) -> u32 {
    a + b
}
```

`trace_me` 的每个调用都会发出一个 `tracing` span，它：

- 具有 [`info` 级别冗余信息](https://docs.rs/tracing/latest/tracing/struct.Level.html)（中间级别）
- 被命名为 `trace_me`
- 具有字段 a 和 b，其值是 `trace_me` 的参数

`instrument` 属性高度可配置，在 [`mini-redis-server`](https://tokio.rs/tokio/tutorial/setup#mini-redis) 中追踪处理每个连接的方法：

```rust
use tracing::instrument;

impl Handler {
    /// Process a single connection.
    #[instrument(
        name = "Handler::run",
        skip(self),
        fields(
            // `%` serializes the peer IP addr with `Display`
            peer_addr = %self.connection.peer_addr().unwrap()
        ),
    )]
    async fn run(&mut self) -> mini_redis::Result<()> {
        ...
    }
}
```

[`mini-redis-server`](https://tokio.rs/tokio/tutorial/setup#mini-redis) 现在将为每个传入连接发出一个 `tracing` span，它：

- 具有 [`info` 级别的冗余信息](https://docs.rs/tracing/latest/tracing/struct.Level.html)（中间级别的详细性）
- 被命名为 `Handler::run`
- 有一些与之相关的结构化数据
	- `fields(...)` 表示发出的 span 应该在名为 `peer_addr` 的字段中包含连接的 `SocketAddr` 的 `fmt::Display` 表示
	- `skip(self)` 表示发出的 span 不应该记录 `Handler` 的调试表示

您还可以通过调用 [`span!`](https://docs.rs/tracing/latest/tracing/#spans) 宏或其任何级别的简写（[`error_span!`](https://docs.rs/tracing/*/tracing/macro.error_span.html)、[`warn_span!`](https://docs.rs/tracing/*/tracing/macro.warn_span.html)、[`info_span!`](https://docs.rs/tracing/*/tracing/macro.info_span.html)、[`debug_span!`](https://docs.rs/tracing/*/tracing/macro.debug_span.html)、[`trace_span!`](https://docs.rs/tracing/*/tracing/macro.trace_span.html)）手动构建 Span。

要发出事件，请调用 [`event!`](https://docs.rs/tracing/*/tracing/macro.event.html) 宏或其任何级别的简写（[`error!`](https://docs.rs/tracing/*/tracing/macro.error.html)、[`warn!`](https://docs.rs/tracing/*/tracing/macro.warn.html)、[`info!`](https://docs.rs/tracing/*/tracing/macro.info.html)、[`debug!`](https://docs.rs/tracing/*/tracing/macro.debug.html)、[`trace!`](https://docs.rs/tracing/*/tracing/macro.trace.html)）。例如，要记录客户端发送了格式错误的命令：

```rust
// Convert the redis frame into a command struct. This returns an
// error if the frame is not a valid redis command.
let cmd = match Command::from_frame(frame) {
    Ok(cmd) => cmd,
    Err(cause) => {
        // The frame was malformed and could not be parsed. This is
        // probably indicative of an issue with the client (as opposed
        // to our server), so we (1) emit a warning...
        //
        // The syntax here is a shorthand provided by the `tracing`
        // crate. It can be thought of as similar to:
        //      tracing::warn! {
        //          cause = format!("{}", cause),
        //          "failed to parse command from frame"
        //      };
        // `tracing` provides structured logging, so information is
        // "logged" as key-value pairs.
        tracing::warn! {
            %cause,
            "failed to parse command from frame"
        };
        // ...and (2) respond to the client with the error:
        Command::from_error(cause)
    }
};
```

如果您运行应用程序，您现在将看到为其处理的每个传入连接发出的以其 span 上下文装饰的事件。

