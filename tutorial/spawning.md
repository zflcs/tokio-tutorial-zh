# Spawning

我们将转换方向并开始研究 Redis 服务器。

首先，将上一节中的客户端 `SET/GET` 代码移至示例文件。这样，我们就可以针对我们的服务器运行它。

```shell
mkdir -p examples
mv src/main.rs examples/hello-redis.rs
```

然后创建一个新的、空的 src/main.rs 并继续。

## Accepting sockets

我们的 Redis 服务器需要做的第一件事是接收入站的 TCP 套接字。这是通过将 tokio::net::TcpListener 绑定到端口 6379 来实现的。

> Tokio 的许多类型都与 Rust 标准库中的同步类型名称相同。在合理的情况下，Tokio 会使用 `async fn` 公开与 `std` 相同的 API。

然后在循环接收套接字。每个套接字在处理完成后被关闭。现在，我们将读取命令，将其打印到 stdout 并以错误响应。

```rust
// src/main.rs
use tokio::net::{TcpListener, TcpStream};
use mini_redis::{Connection, Frame};

#[tokio::main]
async fn main() {
    // Bind the listener to the address
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // The second item contains the IP and port of the new connection.
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}

async fn process(socket: TcpStream) {
    // The `Connection` lets us read/write redis **frames** instead of
    // byte streams. The `Connection` type is defined by mini-redis.
    let mut connection = Connection::new(socket);

    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // Respond with an error
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response).await.unwrap();
    }
}
```

现在，运行这个 accept 循环：

```shell
cargo run
```

在单独的终端窗口中，运行 `hello-redis` 示例（上一节中的 `SET/GET` 命令）：

```shell
cargo run --example hello-redis
```

输出应为：

```shell
Error: "unimplemented"
```

在服务器终端，输出为：

```shell
GOT: Array([Bulk(b"set"), Bulk(b"hello"), Bulk(b"world")])
```

## 并发

我们的服务器有点小问题（除了只响应错误之外）。它每次处理一个入站请求。当连接被接受时，服务器停留在 accept 循环块内，直到响应完全写入套接字。

我们希望 Redis 服务器能够处理许多并发请求。为此，我们需要增加一些并发性。

> 并发和并行不是一回事。如果你在两个任务之间交替，那么你正在并发处理这两个任务，但不是并行的。为了使它符合并行的条件，你需要两个人，一个人专门负责每项任务。
>
> 使用 Tokio 的一个优点是异步代码允许您同时处理多个任务，而无需使用普通线程并行处理它们。事实上，Tokio 可以在单个线程上同时运行多个任务！

为了并发处理连接，每个入站连接都会生成一个新任务。连接在此任务上进行处理。

accept 循环转变为：

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // A new task is spawned for each inbound socket. The socket is
        // moved to the new task and processed there.
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}
```

### 任务

Tokio 任务是异步绿色线程。它们是通过将 `async` 块传递给 `tokio::spawn` 来创建的。`tokio::spawn` 函数返回一个 `JoinHandle`，调用者可以使用它与生成的任务进行交互。`async` 块可能有一个返回值。调用者可以使用 `JoinHandle` 上的 `.await` 获取返回值。

例如：

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Do some async work
        "return value"
    });

    // Do some other work

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

`JoinHandle` 上的等待将返回一个结果。当任务在执行过程中遇到错误时，`JoinHandle` 将返回一个 `Err`。当任务崩溃或运行时关闭强制取消任务时，就会发生这种情况。

任务是调度程序管理的执行单位。生成任务将其提交给 Tokio 调度程序，然后调度程序确保在有工作要做时执行该任务。生成的任务可能在生成的同一线程上执行，也可能在不同的运行时线程上执行。生成任务后，还可以在线程之间移动。

Tokio 中的任务非常轻量。在底层，它们只需要一次分配和 64 字节内存。应用程序可以自由地生成数千个甚至数百万个任务。

### `'static` 约束 

当您在 Tokio 运行时上生成任务时，其类型的生存期必须是 `'static`。这意味着生成的任务不得包含对任务之外拥有的数据的任何引用。

> 一个常见的误解是，`'static` 总是意味着“永远存在”，但事实并非如此。仅仅因为一个值是 `'static` 并不意味着存在内存泄漏。您可以在 [Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program) 中阅读更多内容。

例如，下面的代码将无法编译：

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

尝试编译此结果会导致以下错误：

```shell
error[E0373]: async block may outlive the current function, but
              it borrows `v`, which is owned by the current function
 --> src/main.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:7:17
  |
7 |       task::spawn(async {
  |  _________________^
8 | |         println!("Here's a vector: {:?}", v);
9 | |     });
  | |_____^
help: to force the async block to take ownership of `v` (and any other
      referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |
```

发生这种情况的原因是，默认情况下变量不会移动到异步块中。`v` 向量仍归 `main` 函数所有。`println!` 行借用了 `v`。rust 编译器很有帮助地向我们解释了这一点，甚至建议了修复方法！将第 ​​7 行更改为 `task::spawn(async move {` 将指示编译器将 `v` 移动到生成的任务中。现在，该任务拥有其所有数据，使其成为 `'static`。

如果必须同时从多个任务访问单个数据，则必须使用同步原语（如 `Arc`）来共享该数据。

请注意，错误消息指出参数类型的生命周期超过了 `'static`。这个术语可能相当令人困惑，因为 `'static` 持续到程序结束，因此如果它比程序结束还长，那么不会有内存泄漏吗？解释是它是类型，而不是 值必须比 `'static` 生命周期长，并且该值可以在其类型不再有效之前被销毁。

当我们说某个值是 `'static` 时，这意味着永远保留该值不会是错误的。这很重要，因为编译器无法推断新生成的任务会保留多长时间。我们必须确保任务能够永远存在，以便 Tokio 可以让任务运行到需要的时间。

info-box 前面链接到的文章使用术语 “由 `'static` 约束”，而不是 “其类型超出 `'static`” 或 “值是 `'static` ” 来引用 `T: 'static` 。这些都意味着同一件事，但与 `&'static T` 中的 “用 `'static` 注释”不同。

### `Send` 约束

由 `tokio::spawn` 生成的任务必须实现 `Send`。这允许 Tokio 运行时在 `.await` 处暂停任务时在线程之间移动任务。

当跨越 `.await` 调用的所有数据都实现了 `Send` 时，整个任务才是满足 `Send` 约束。这有点微妙。调用 `.await` 时，任务将让权给调度程序。下次执行该任务时，它会从上次让权的点继续执行。为了使其工作，`.await` 之后使用的所有状态都必须由任务保存。如果此状态为 `Send`，即可跨线程移动，则任务本身也可以跨线程移动。

相反，如果状态不是 `Send`，则任务也不是。

例如，这是有效的：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // The scope forces `rc` to drop before `.await`.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` is no longer used. It is **not** persisted when
        // the task yields to the scheduler
        yield_now().await;
    });
}
```

这不会：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` is used after `.await`. It must be persisted to
        // the task's state.
        yield_now().await;

        println!("{}", rc);
    });
}
```

尝试编译该代码片段会导致：

```shell
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    | 
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

## Store values

我们现在将实现 `process` 函数来处理传入的命令。我们将使用 `HashMap` 来存储值。 `SET` 命令将插入到 `HashMap`，`GET` 将获取它们。此外，我们将使用循环来为每个连接接受多个命令。

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream) {
    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // A hashmap is used to store data
    let mut db = HashMap::new();

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    // Use `read_frame` to receive a command from the connection.
    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                // The value is stored as `Vec<u8>`
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    // `Frame::Bulk` expects data to be of type `Bytes`. This
                    // type will be covered later in the tutorial. For now,
                    // `&Vec<u8>` is converted to `Bytes` using `into()`.
                    Frame::Bulk(value.clone().into())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // Write the response to the client
        connection.write_frame(&response).await.unwrap();
    }
}
```

现在，启动服务器：

```shell
cargo run
```

并在单独的终端窗口中运行 `hello-redis` 示例：

```shell
cargo run --example hello-redis
```

现在，输出将是：

```shell
got value from the server; result=Some(b"world")
```

我们现在可以获取和设置值，但有一个问题：连接之间不共享这些值。如果另一个套接字连接并尝试 `GET` `hello` 键，它不会找到任何东西。

您可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/spawning/src/main.rs)找到完整的代码。

在下一节中，我们将为所有套接字实现持久数据。
