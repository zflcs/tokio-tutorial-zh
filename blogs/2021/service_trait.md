# Inventing the Service trait

[Tower](https://github.com/tower-rs/tower) 是一个模块化和可重复使用的组件库，用于构建强大的网络客户端和服务器。其核心是 [`Service`](https://docs.rs/tower/latest/tower/trait.Service.html) trait。`Service` 是一种异步函数，它接受请求并产生响应。然而，其设计的某些方面可能并不明显。与其解释 Tower 中现有的 `Service` trait，不如让我们通过想象如果你从头开始，你将如何发明它来了解 `Service` 背后的动机。

假设你正在用 Rust 构建一个小型 HTTP 框架。该框架将允许用户通过提供接收请求并回复某些响应的代码来实现 HTTP 服务器。

你可能有一个如下的 API：

```rust
// Create a server that listens on port 3000
let server = Server::new("127.0.0.1:3000").await?;

// Somehow run the user's application
server.run(the_users_application).await?;
```

问题是，`the_users_application` 应该是什么？

最简单的可能有效的方法是：

```rust
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}
```

其中 `HttpRequest` 和 `HttpResponse` 是我们框架提供的一些结构。

有了这个，我们可以像这样实现 `Server::run`：

```rust
impl Server {
    async fn run<F>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> HttpResponse,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Call the handler provided by the user
                let response = handler(request);

                write_http_response(connection, response).await?;
            });
        }
    }
}
```

这里，我们有一个异步函数 `run`，它的参数为一个闭包，该闭包以 `HttpRequest` 作为参数并返回 `HttpResponse` 。

这意味着用户可以像这样使用我们的 `Server`：

```rust
fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else {
        HttpResponse::not_found()
    }
}

// Run the server and handle requests using our `handle_request` function
server.run(handle_request).await?;
```

这还不错。它让用户可以轻松运行 HTTP 服务器，而不必担心任何底层细节。

然而，我们目前的设计有一个问题：我们无法异步处理请求。想象一下，我们的用户需要在处理请求的同时查询数据库或向其他服务器发送请求。目前，这需要我们在等待处理程序产生响应时进行[阻塞](https://ryhl.io/blog/async-what-is-blocking/)。如果我们希望服务器能够处理大量并发连接，我们需要能够在等待该请求异步完成的同时处理其他请求。让我们通过让处理程序函数返回一个 [future](https://doc.rust-lang.org/stable/std/future/trait.Future.html) 来解决这个问题：

```rust
impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        // `handler` now returns a generic type `Fut`...
        F: Fn(HttpRequest) -> Fut,
        // ...which is a `Future` whose `Output` is an `HttpResponse`
        Fut: Future<Output = HttpResponse>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Await the future returned by `handler`
                let response = handler(request).await;

                write_http_response(connection, response).await?;
            });
        }
    }
}
```

此 API 的使用与之前非常相似：

```rust
// Now an async function
async fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else if request.path() == "/important-data" {
        // We can now do async stuff in here
        let some_data = fetch_data_from_database().await;
        make_response(some_data)
    } else {
        HttpResponse::not_found()
    }
}

// Running the server is the same
server.run(handle_request).await?;
```

这要好得多，因为我们的请求处理现在可以调用其他异步函数。但是，仍然缺少一些东西。如果我们的处理程序遇到错误并且无法产生响应怎么办？让我们让它返回一个`Result`：

```rust
impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> Fut,
        // The response future is now allowed to fail
        Fut: Future<Output = Result<HttpResponse, Error>>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Pattern match on the result of the response future
                match handler(request).await {
                    Ok(response) => write_http_response(connection, response).await?,
                    Err(error) => handle_error_somehow(error, connection),
                }
            });
        }
    }
}
```

## Adding more behavior

现在，假设我们想确保所有请求及时完成或失败，而不是让客户端无限期地等待可能永远不会到达的响应。我们可以通过为每个请求添加超时来实现这一点。超时设置了处理程序允许的最大持续时间限制。如果它在该时间内未产生响应，则会返回错误。这允许客户端重试该请求或向用户报告错误，而不是永远等待。

您的第一个想法可能是修改 `Server`，以便可以配置超时。然后每次调用处理程序时都会应用该超时。然而，事实证明您实际上可以在不修改 `Server` 的情况下添加超时。使用 [`tokio::time::timeout`](https://docs.rs/tokio/latest/tokio/time/fn.timeout.html)，我们可以创建一个新的处理函数，调用我们之前的 `handle_request`，但超时为 30 秒：

```rust
async fn handler_with_timeout(request: HttpRequest) -> Result<HttpResponse, Error> {
    let result = tokio::time::timeout(
        Duration::from_secs(30),
        handle_request(request)
    ).await;

    match result {
        Ok(Ok(response)) => Ok(response),
        Ok(Err(error)) => Err(error),
        Err(_timeout_elapsed) => Err(Error::timeout()),
    }
}
```


