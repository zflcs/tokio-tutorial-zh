# Diagnostics with Tracing

有效地开发系统并在生产中运行它们需要了解它们在运行时的行为。虽然传统日志记录可以提供部分可见性，但异步软件（如使用 Tokio 运行时的应用程序）带来了新的挑战。

[`tracing`](https://crates.io/crates/tracing) 是一个库的集合，它为检测 Rust 程序以收集结构化、上下文感知、事件驱动的诊断信息提供了一个框架。请注意，`tracing` 最初是以名称 `tokio-trace` 发布的；名称已更改以反映尽管它是 Tokio 项目的一部分，但 `tokio` 运行时不要求使用 `tracing`。

## Why do we need another logging library?

Rust 已经拥有一个基于 [log](https://crates.io/crates/log) crate 的日志记录外观的强大日志记录生态系统，其中 `env_logger` 和 `fern` 等库被广泛使用。这引发了一些合理的问题——为什么 `tracing` 是必要的，它提供了哪些现有库所没有的好处？为了回答这些问题，我们需要考虑异步系统中诊断带来的挑战。

在同步代码中，我们可以简单地在程序执行时记录单个消息，并期望它们按顺序打印。程序员可以相当轻松地解释日志，因为日志记录是按顺序输出的。例如，在同步系统中，如果我看到一组如下的日志消息：

```shell
DEBUG server: accepted connection from 106.42.126.8:56975
DEBUG server::http: received request
 WARN server::http: invalid request headers
DEBUG server: closing connection
```

我可以推断，来自 IP 地址为 106.42.126.8 的客户端的请求是失败的请求，并且来自该客户端的连接随后被服务器关闭。先前的消息暗示了上下文：因为同步服务器必须在接受下一个连接之前处理每个请求，所以我们可以确定在“接受连接...”消息之后和“关闭连接”消息之前出现的任何日志记录都指的是该连接。

然而，在像 Tokio 这样的异步系统中，解释传统的日志消息通常非常具有挑战性。异步系统中的单个线程可能执行任意数量的任务，并在 IO 资源可用时在它们之间切换，并且应用程序可能由多个同时运行的此类工作线程组成。在这个世界上，我们不能再依赖日志消息的顺序来确定上下文或因果关系。任务可能会记录一些消息并让权，从而允许 executor 轮询另一个记录其自身不相关消息的任务。来自并发运行的线程的日志消息可能会交错打印出来。为了理解异步系统，必须明确记录上下文和因果关系，而不是通过顺序来暗示。

如果上述示例中的日志行是由异步应用​​程序发出的，则服务器任务可能会继续接受新的连接，而先前接受的连接正在由其他任务处理；多个请求可能会被工作线程同时处理。我们可能会看到类似这样的情况：

```shell
DEBUG server: accepted connection from 106.42.126.8:56975
DEBUG server: closing connection
DEBUG server::http: received request
DEBUG server: accepted connection from 11.103.8.9:49123
DEBUG server::http: received request
DEBUG server: accepted connection from 102.12.37.105:51342
 WARN server::http: invalid request headers
TRACE server: closing connection
```

我们不知道是否还从 106.42.126.8 接收到了带有无效标头的请求。由于同时处理多个连接，因此当我们等待从该客户端接收更多数据时，无效标头可能已在另一个连接上发送。我们该怎么做才能理清这一混乱局面？

传统的日志记录通常会捕获有关程序的静态上下文 - 例如记录事件的文件、模块或函数 - 但这对我们理解程序的运行时行为用处有限。相反，异步代码的可见性需要跟踪事件发生的动态运行时上下文的诊断。

## Application-Level Tracing

`tracing` 不仅仅是一个日志库：它实现了范围、上下文和结构化的诊断工具。这使得用户能够随时间跟踪应用程序中的逻辑上下文，即使实际的执行流在这些上下文之间移动。

## Spans

为了记录执行流程，`tracing` 引入了 span 的概念。与表示时间点的日志行不同，span 模拟了具有开始和结束的一段时间。当程序开始在上下文中执行或执行工作单元时，它会进入 span；当它停止在该上下文中执行时，它会退出 span。

在进入 span 和退出 span 之间发生的任何事件都被视为在该 span 内发生。类似地，span 可以嵌套：当一个线程进入另一个 span 内的 span 时，它将同时处于这两个 span 中，新进入的 span 被视为子 span，而外部 span 被视为父 span。然后，我们可以构建一个嵌套 span 树，并在程序的不同部分中跟踪它们。

`tracing` 还支持事件，事件可对瞬时时间点进行建模。事件类似于传统的日志消息，但也存在于 span 树中。当我们记录事件时，我们可以精确定位事件发生的上下文。

## Structure

通过将结构化数据附加到 span，我们可以对上下文进行建模。`tracing` 检测点不是简单地记录非结构化、人类可读的消息，而是记录类型化的键值数据（称为字段）。例如，在 HTTP 服务器中，表示接受连接的 span 可能会记录客户端的 IP 地址、请求的路径、请求方法、标头等字段。如果我们重新审视上面的例子，并添加 span，我们可能会看到如下内容：

```shell
DEBUG server{client.addr=106.42.126.8:56975}: accepted connection
DEBUG server{client.addr=82.5.70.2:53121}: closing connection
DEBUG server{client.addr=89.56.1.12:55601} request{path="/posts/tracing" method=GET}: received request
DEBUG server{client.addr=111.103.8.9:49123}: accepted connection
DEBUG server{client.addr=106.42.126.8:56975} request{path="/" method=PUT}: received request
DEBUG server{client.addr=113.12.37.105:51342}: accepted connection
 WARN server{client.addr=106.42.126.8:56975} request{path="/" method=PUT}: invalid request headers
TRACE server{client.addr=106.42.126.8:56975} request{path="/" method=PUT}: closing connection
```

请注意事件是如何用记录客户端 IP 地址以及请求的路径和方法的 span 进行注释的。尽管多个事件在不同的上下文中同时发生，我们现在可以跟踪来自 106.42.126.8 的请求在系统中的流程，并确定是包含无效标头的请求产生了警告。

这种机器可读的结构化数据还使我们能够以更复杂的方式使用诊断数据，而不是简单地将其格式化以供人类阅读。例如，我们还可以通过计算不同路径或 HTTP 方法收到的请求数来使用上述数据。通过查看 span 树的结构以及键值数据，我们甚至可以做一些事情，例如，只有当请求以错误结束时才记录请求的整个生命周期。

## A Worked Example

为了展示这种诊断的价值，我们来看一个例子。在这个例子中，我们使用 [`tower`](https://crates.io/crates/tower) 编写了一个小型 HTTP 服务，实现了“字符重复即服务”。该服务接收由单个 ASCII 字符组成的路径请求，并以该字符的重复次数进行响应，该次数等于 `Content-Length` 标头的值。

我们已经使用 `tracing` 功能对示例服务进行了检测，并使用 [`tracing-subscriber`](https://crates.io/crates/tracing-subscriber) crate 实现了管理端点，该端点允许我们更改确定在运行时启用哪些 `tracing` 检测的[过滤器配置](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/reload/index.html)。负载生成器在后台运行，不断向演示服务发送随机字母字符的请求。

使用 [`log`](https://crates.io/crates/log) 和 [`env_logger`](https://crates.io/crates/env_logger) crates 检测示例服务版本后，我们得到如下输出：

```shell
...
[DEBUG load_log] accepted connection from [::1]:55257
[DEBUG load_log] received request for path "/z"
[DEBUG load_log] accepted connection from [::1]:55258
[DEBUG load_log] received request for path "/Z"
[ERROR load_log] error received from server! status: 500
[DEBUG load_log] accepted connection from [::1]:55259
[DEBUG load_log] accepted connection from [::1]:55260
[DEBUG load_log] received request for path "/H"
[DEBUG load_log] accepted connection from [::1]:55261
[DEBUG load_log] received request for path "/S"
[DEBUG load_log] received request for path "/C"
[DEBUG load_log] accepted connection from [::1]:55262
[DEBUG load_log] received request for path "/x"
[DEBUG load_log] accepted connection from [::1]:55263
[DEBUG load_log] accepted connection from [::1]:55264
[WARN  load_log] path "/" not found; returning 404
[DEBUG load_log] accepted connection from [::1]:55265
[ERROR load_log] error received from server! status: 404
[DEBUG load_log] received request for path "/Q"
[DEBUG load_log] accepted connection from [::1]:55266
[DEBUG load_log] received request for path "/l"
...
```

由于服务处于负载状态，这些消息滚动得非常快。我在这里只提供了一小部分输出示例。

尽管错误确实出现在日志中，但很难将它们与任何可能帮助我们确定其原因的背景联系起来。对于 404 错误，我们足够幸运，因为添加服务器记录的警告行的人认为将路径包含在人类可读的日志消息中，但对于 500 错误，我们只能盲目操作。

现在，让我们切换到带有 `tracing` 功能的演示应用程序版本：

```shell
...
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60891}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60890}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60891}:request{req.method=GET req.path="/I"}: load: received request. req.headers={"content-length": "18", "host": "[::1]:3000"} req.version=HTTP/1.1
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60890}:request{req.method=GET req.path="/T"}: load: received request. req.headers={"content-length": "4", "host": "[::1]:3000"} req.version=HTTP/1.1
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60892}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60892}:request{req.method=GET req.path="/x"}: load: received request. req.headers={"content-length": "6", "host": "[::1]:3000"} req.version=HTTP/1.1
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60893}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60895}:request{req.method=GET req.path="/"}: load: received request. req.headers={"content-length": "13", "host": "[::1]:3000"} req.version=HTTP/1.1
 WARN server{name=serve local=[::1]:3000}:conn{remote=[::1]:60895}:request{req.method=GET req.path="/"}: load: rsp.status=404
ERROR gen: error received from server! status=404
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60893}:request{req.method=GET req.path="/a"}: load: received request. req.headers={"content-length": "11", "host": "[::1]:3000"} req.version=HTTP/1.1
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60894}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60894}:request{req.method=GET req.path="/V"}: load: received request. req.headers={"content-length": "12", "host": "[::1]:3000"} req.version=HTTP/1.1
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60896}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60998}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60996}:request{req.method=GET req.path="/z"}: load: received request. req.headers={"content-length": "17", "host": "[::1]:3000"} req.version=HTTP/1.1
ERROR gen: error received from server! status=500
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60987}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60911}: load: accepted connection
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60911}:request{req.method=GET req.path="/m"}: load: received request. req.headers={"content-length": "7", "host": "[::1]:3000"} req.version=HTTP/1.1
DEBUG server{name=serve local=[::1]:3000}:conn{remote=[::1]:60912}: load: accepted connection
...
```

请注意，除了打印单个事件外，还会打印发生事件的 span 上下文。特别是，每个连接和请求都会创建自己的 span。但是，这些事件记录得非常频繁，所以日志仍然滚动得相当快。

如果我们向演示应用程序的管理端点发送请求，我们可以重新加载过滤器配置以仅查看负载生成器记录的事件：

```shell
:; curl -X PUT localhost:3001/filter -d "gen=debug"
```

这些过滤器使用与 [`env_logger`](https://crates.io/crates/env_logger) crate 类似的语法指定。日志滚动的速度明显减慢，我们看到如下输出：

```shell
...
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
...
```

查看负载生成器输出的 `request` span，我们开始注意到一个模式。`req.path` 字段的值始终为 `"/"` 或 `"/z"`。

我们可以再次重新加载过滤器配置，仅当我们位于字段 `req.path` 的值为 `"/"` 的范围内时，才将详细程度设置为最大值：

```shell
:; curl -X PUT localhost:3001/filter -d "[{req.path=\"/\"}]=trace"
```

`[]` 表示我们希望过滤跟踪 span 而不是静态目标，而 `{}` 表示我们想要匹配 span 字段。

```shell
...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: sending request...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: tower_buffer::service: sending request to buffer worker
DEBUG request{req.method=GET req.path="/"}: load: received request. req.headers={"content-length": "21", "host": "[::1]:3000"} req.version=HTTP/1.1
TRACE request{req.method=GET req.path="/"}: load: handling request...
TRACE request{req.method=GET req.path="/"}: load: rsp.error=path must be a single ASCII character
 WARN request{req.method=GET req.path="/"}: load: rsp.status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: response complete. rsp.body=path must be a single ASCII character
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: sending request...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: tower_buffer::service: sending request to buffer worker
DEBUG request{req.method=GET req.path="/"}: load: received request. req.headers={"content-length": "18", "host": "[::1]:3000"} req.version=HTTP/1.1
TRACE request{req.method=GET req.path="/"}: load: handling request...
TRACE request{req.method=GET req.path="/"}: load: rsp.error=path must be a single ASCII character
 WARN request{req.method=GET req.path="/"}: load: rsp.status=404
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: error received from server! status=404
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: response complete. rsp.body=path must be a single ASCII character
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: gen: sending request...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/"}: tower_buffer::service: sending request to buffer worker
DEBUG request{req.method=GET req.path="/"}: load: received request. req.headers={"content-length": "2", "host": "[::1]:3000"} req.version=HTTP/1.1
...
```

现在我们明白发生了什么。该服务不支持对 `/` 的请求，并且正确地响应了 404。但是那些 500 错误怎么办？我们记得它们似乎只在请求的路径为 "/z" 时才会发生……

```shell
:; curl -X PUT localhost:3001/filter -d "[{req.path=\"/z\"}]=trace"
```

```shell
...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: sending request...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: tower_buffer::service: sending request to buffer worker
DEBUG request{req.method=GET req.path="/z"}: load: received request. req.headers={"content-length": "0", "host": "[::1]:3000"} req.version=HTTP/1.1
TRACE request{req.method=GET req.path="/z"}: load: handling request...
TRACE request{req.method=GET req.path="/z"}: load: error=i don't like this letter. letter="z"
TRACE request{req.method=GET req.path="/z"}: load: rsp.error=unknown internal error
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: response complete. rsp.body=unknown internal error
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: sending request...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: tower_buffer::service: sending request to buffer worker
DEBUG request{req.method=GET req.path="/z"}: load: received request. req.headers={"content-length": "16", "host": "[::1]:3000"} req.version=HTTP/1.1
TRACE request{req.method=GET req.path="/z"}: load: handling request...
TRACE request{req.method=GET req.path="/z"}: load: error=i don't like this letter. letter="z"
TRACE request{req.method=GET req.path="/z"}: load: rsp.error=unknown internal error
ERROR load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: error received from server! status=500
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: response complete. rsp.body=unknown internal error
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: gen: sending request...
TRACE load_gen{remote.addr=[::1]:3000}:request{req.method=GET req.path="/z"}: tower_buffer::service: sending request to buffer worker
DEBUG request{req.method=GET req.path="/z"}: load: received request. req.headers={"content-length": "24", "host": "[::1]:3000"} req.version=HTTP/1.1
TRACE request{req.method=GET req.path="/z"}: load: handling request...
TRACE request{req.method=GET req.path="/z"}: load: error=i don't like this letter. letter="z"
TRACE request{req.method=GET req.path="/z"}: load: rsp.error=unknown internal error
...
```

我们现在可以追踪对 `/z` 请求的整个生命周期，忽略同时发生的所有其他事情。而且，查看记录的事件，我们发现服务记录了“我不喜欢这封信”并返回了内部错误。我们发现了这个（诚然，完全是假的）错误！

这种动态过滤只有通过上下文感知的结构化诊断仪器才有可能实现。因为嵌套 span 和事件可以让我们构建相关上下文的树，我们可以根据逻辑上或因果上相关的事件（例如，它们在处理同一请求时发生）而不是时间上相关的事件（它们在时间上彼此接近）来解释诊断。

如果您有兴趣查看生成这些日志的代码，或者以交互方式试验此演示，您可以查看[此处](https://github.com/tokio-rs/tracing/blob/master/examples/examples/tower-load.rs)的示例。

## Getting Started with Tracing

可以在 [crates.io](https://crates.io/crates/tracing) 上获取 `tracing`：

```rust
tracing = "0.1.5"
```

开始 `tracing` 的最简单方法是使用函数上的 [`tracing::instrument`](https://docs.rs/tracing-attributes/latest/tracing_attributes/attr.instrument.html) 属性。此属性将检测函数，以便在调用该函数时创建并输入新的span，并将函数的参数记录为该 span 上的字段。例如：

```rust
use tracing::instrument;

#[instrument]
pub async fn connect_to(remote: SocketAddr) -> io::Result<TcpStream> {
    // ...
}
```

`tracing` 还提供了一组[类似函数的宏](https://docs.rs/tracing/0.1.5/tracing/#macros)，用于构建 span 和事件。使用 `log` crate 的用户应该注意，`tracing` 的 `trace!`、`debug!`、`info!`、`warn!` 和 `error!` 宏是 `log` 中类似名称的宏的超集，并且应该兼容：

```rust
use log::info;

info!("hello world!");
```

```rust
use tracing::info;

info!("hello world!");
```

然而，更惯用的风格是使用这些宏来记录结构化数据而不是非结构化消息。例如：

```rust
use tracing::trace;

let bytes_read = ...;
let num_processed = ...;

// ...

trace!(bytes_read, messages = num_processed);
```

[Subscriber](https://docs.rs/tracing/0.1.5/tracing/trait.Subscriber.html) 实现会收集并记录跟踪数据，类似于传统日志记录中的记录器。应用程序必须设置[default subscriber](https://docs.rs/tracing/0.1.5/tracing/dispatcher/index.html#setting-the-default-subscriber)。

`Subscriber` 接口是 `tracing` 的主要扩展点；记录和处理跟踪数据的不同方法和策略可以表示为 `Subscriber` 实现。目前，[`tracing-fmt`](https://crates.io/crates/tracing-fmt/) crate 提供了一个将跟踪数据记录到控制台的 subscriber 实现，并且很快会有更多的实现。

更多 [API 文档](https://docs.rs/tracing/0.1.5/tracing/)可在 docs.rs 上找到，并且 [`tracing` github 存储库](https://github.com/tokio-rs/tracing)中提供了示例。

## Building an Ecosystem

`tracing` 生态系统以 [`tracing`](https://crates.io/crates/tracing) crate 为中心，它提供了用于检测库和应用程序的 API，以及 [`tracing-core`](https://crates.io/crates/tracing-core)，它提供了将该仪器与 `subscriber` 连接所需的最小且稳定的功能内核。然而，这只是冰山一角。 [`tokio-rs/tracing`](https://github.com/tokio-rs/tracing) 存储库包含许多其他 crate，稳定性各不相同。这些 crate 包括：

- 与其他库的兼容层，例如 [`tracing-tower`](https://github.com/tokio-rs/tracing/blob/master/tracing-tower) 和 [`tracing-log`](https://crates.io/crates/tracing-log)。
- `Subscriber` 实现，例如 [`tracing-fmt`](https://crates.io/crates/tracing-fmt)。
- [`tracing-subscriber`](https://crates.io/crates/tracing-subscriber)，提供用于实现和组成 `Subscriber` 的工具。

中心 crate 的稳定版本已发布到 crates.io，并且 `tracing` 功能已被 [Linkerd 2](https://github.com/linkerd/linkerd2-proxy) 和 [Vector](https://github.com/timberio/vector) 等项目采用。然而，未来还有很多工作要做，包括：

- 与 [OpenTelemetry](https://opentelemetry.io/) 或 [Jaeger](https://www.jaegertracing.io/) 等分布式跟踪系统集成。
- 在 Tokio 运行时构建更丰富的工具。
- 与更多库和框架集成。
- 编写 `Subscriber` 来实现更多收集跟踪数据的方式，例如指标、分析等等。
- 帮助稳定实验 crates。

所有这些领域的贡献都将受到热烈欢迎。我们都期待看到社区将在 `tracing` 提供的平台上构建什么！

如果您有兴趣，请查看 [GitHub 上](https://github.com/tokio-rs/tracing)的 `tracing` 或加入 [Gitter](https://gitter.im/tokio-rs/tracing) 聊天频道！
