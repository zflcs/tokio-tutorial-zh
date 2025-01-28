# Select

到目前为止，当我们想要向系统添加并发性时，我们生成了一个新任务。我们现在将介绍一些使用 Tokio 并发执行异步代码的其他方法。

## `tokio::select!`

`tokio::select!` 宏允许等待多个异步计算并在单个计算完成时返回。

例如：

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        let _ = tx1.send("one");
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

使用两个 oneshot 通道。任一通道都可以先完成。这 `select!` 语句在两个通道上等待并将 val 绑定到任务返回的值。当 tx1 或 tx2 完成时，执行关联的块。

未完成的分支将被删除。在示例中，计算正在等待每个通道的 `oneshot::Receiver`。这尚未完成的通道的 `oneshot::Receiver` 将被丢弃。

## Cancellation

对于异步 Rust，取消是通过删除 future 来执行的。回想一下 [Async in depth](./async_in_depth.md)，异步 Rust 操作是使用 future 实现的，而 future 是惰性的。该操作仅在 future 被 poll 时执行。如果 future 被删除，操作将无法继续，因为所有关联的状态都已被删除。

也就是说，有时异步操作会生成后台任务或启动在后台运行的其他操作。例如，在上面的示例中，生成了一个任务来发回消息。通常，任务将执行一些计算来生成值。

Futures 或者其他类型可以实现 `Drop` 来清理后台资源。Tokio 的 `oneshot::Receiver` 通过向 Sender 端发送关闭通知来实现 `Drop`。Sender 端可以接收此通知并通过 drop 来中止正在进行的操作。

```rust
use tokio::sync::oneshot;

async fn some_operation() -> String {
    // Compute value here
}

#[tokio::main]
async fn main() {
    let (mut tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    tokio::spawn(async {
        // Select on the operation and the oneshot's
        // `closed()` notification.
        tokio::select! {
            val = some_operation() => {
                let _ = tx1.send(val);
            }
            _ = tx1.closed() => {
                // `some_operation()` is canceled, the
                // task completes and `tx1` is dropped.
            }
        }
    });

    tokio::spawn(async {
        let _ = tx2.send("two");
    });

    tokio::select! {
        val = rx1 => {
            println!("rx1 completed first with {:?}", val);
        }
        val = rx2 => {
            println!("rx2 completed first with {:?}", val);
        }
    }
}
```

### The `Future` implementaion

为了帮助大家更好的理解 `select!` 如何有效，让我们看看假设的 `Future` 实现会是什么样子。这是一个简化版本。实际操作中， `select!` 包括附加功能，例如随机选择要首先 poll 的分支。

```rust
use tokio::sync::oneshot;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct MySelect {
    rx1: oneshot::Receiver<&'static str>,
    rx2: oneshot::Receiver<&'static str>,
}

impl Future for MySelect {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if let Poll::Ready(val) = Pin::new(&mut self.rx1).poll(cx) {
            println!("rx1 completed first with {:?}", val);
            return Poll::Ready(());
        }

        if let Poll::Ready(val) = Pin::new(&mut self.rx2).poll(cx) {
            println!("rx2 completed first with {:?}", val);
            return Poll::Ready(());
        }

        Poll::Pending
    }
}

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    // use tx1 and tx2

    MySelect {
        rx1,
        rx2,
    }.await;
}
```

`MySelect` future 包含来自每个分支的 future。当 `MySelect` 为 polled，第一个分支被 poll。如果它是 ready，则使用该值且 `MySelect` 完成。当 `.await` 收到 future 的输出后，future 就会被丢弃。这会导致两个分支的 future 都被放弃。由于一个分支未完成，操作实际上被取消。

请记住上一节中的内容：

> 当 future 返回 `Poll::Pending` 时，它必须确保 waker 在未来的某个时刻收到信号。忘记执行此操作会导致任务无限期挂起。

`MySelect` 中没有明确使用 `Context` 参数来执行。相反，waker 要求是通过将 `cx` 传递到内部 future 来满足的。由于内部 future 也必须满足 waker 要求，因此仅在从内部 future 接收 `Poll::Pending` 时返回 `Poll::Pending`，`MySelect` 仍然满足 Waker 的要求。

## Syntax

`select!` 宏可以处理两个以上的分支。当前限制为64个分支。每个分支的结构为：

```rust
<pattern> = <async expression> => <handler>,
```

当 `select` 宏被解析时，所有 `<async expression>` 均被汇总且并发执行。当表达式完成后，结果与 `<pattern>` 匹配。如果结果与模式匹配，则将删除所有剩余的异步表达式，并执行 `<handler>`。这 `<handler>` 表达式可以访问 `<pattern>` 建立的任何绑定。

`<pattern>` 的基本类型是一个变量名称，异步表达式的结果与变量名称绑定， `<handler>` 可以访问该变量。这就是为什么在原始示例中 `<pattern>` 使用 `val` 以及 `<handler>` 能够访问 val 的原因。

如果 `<pattern>` 与异步计算的结果不符，则其余异步表达式继续并发执行，直到下一个表达式完成为止，且采用相同的逻辑应用于该结果。

因为 `select!` 可以接收任何异步表达式，可以在 select 上定义更复杂的计算。

在这里，我们对 `oneshot` 通道以及 TCP 连接的输出使用 select。

```rust
use tokio::net::TcpStream;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    // Spawn a task that sends a message over the oneshot
    tokio::spawn(async move {
        tx.send("done").unwrap();
    });

    tokio::select! {
        socket = TcpStream::connect("localhost:3465") => {
            println!("Socket connected {:?}", socket);
        }
        msg = rx => {
            println!("received message first {:?}", msg);
        }
    }
}
```

在这里，我们针对 oneshot 使用 select，并接收来自 TcpListener 的套接字。

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        tx.send(()).unwrap();
    });

    let mut listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        _ = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {}
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
```

接收循环运行，直到遇到错误或 `rx` 收到值。`_` 模式表明我们对异步计算的返回值没有兴趣。

## Return value

`tokio::select!` 宏返回 `<handler>` 表达式的结果。

```rust
async fn computation1() -> String {
    // .. computation
}

async fn computation2() -> String {
    // .. computation
}

#[tokio::main]
async fn main() {
    let out = tokio::select! {
        res1 = computation1() => res1,
        res2 = computation2() => res2,
    };

    println!("Got = {}", out);
}
```

因此，要求每种 `<handler>` 表达式分支返回相同的类型。如果 `select!` 的输出不需要，最好将表达式返回为 `()`。

## Errors

使用 `?` 操作符从表达式中传递错误，这取决于异步表达式或 handler 中是否使用 `?`。异步表达式中使用 `?` 用于向外部传递错误，使得输出成为 `Result`。在 handler 中使用 `?` 会将错误立即传递给 `select!`。让我们再次查看接收循环的示例：

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    // [setup `rx` oneshot channel]

    let listener = TcpListener::bind("localhost:3465").await?;

    tokio::select! {
        res = async {
            loop {
                let (socket, _) = listener.accept().await?;
                tokio::spawn(async move { process(socket) });
            }

            // Help the rust type inferencer out
            Ok::<_, io::Error>(())
        } => {
            res?;
        }
        _ = rx => {
            println!("terminating accept loop");
        }
    }

    Ok(())
}
```

注意到 `listener.accept().await?`。`?` 操作符将错误从该表达式传递到 `res` 绑定。发生错误时， `res` 将被设置为 `Err(_)`。然后，在 handler 中再次使用 `?` 操作符。`res?` 语句会将错误传递到 `main` 函数之外。

## Pattern matching

回想一下，`select!` 宏分支语法定义为：

```rust
<pattern> = <async expression> => <handler>,
```

到目前为止，我们仅使用了 `<pattern>` 的变量绑定。但是，任何 Rust 的模式匹配都可以使用。例如，假设我们从多个 MPSC 通道接收的，我们可能会做这样的事情：

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (mut tx1, mut rx1) = mpsc::channel(128);
    let (mut tx2, mut rx2) = mpsc::channel(128);

    tokio::spawn(async move {
        // Do something w/ `tx1` and `tx2`
    });

    tokio::select! {
        Some(v) = rx1.recv() => {
            println!("Got {:?} from rx1", v);
        }
        Some(v) = rx2.recv() => {
            println!("Got {:?} from rx2", v);
        }
        else => {
            println!("Both channels closed");
        }
    }
}
```

在此示例中，`select!` 表达式等待从 `rx1` 和 `rx2` 接收值。如果通道关闭，则 `recv()` 返回 `None`。这与模式不匹配，并且分支无效。 `select!` 将继续等待其余分支。

注意这个 `select!` 表达式包含 `else` 分支。 `select!` 表达式必须计算出一个值。当使用模式匹配时，允许没有分支与其关联的模式相匹配。如果发生这种情况，则会执行 `else` 分支。

## Borrowing

生成任务时，异步表达式必须拥有其所有数据。而 `select!` 宏没有这个限制。每个分支的异步表达式可以借用数据且并发操作。遵循 Rust 的借用规则，多个异步表达式可以不可变地借用同一个数据，或者单个异步表达式可以可变地借用一个数据。

让我们看一些例子。在这里，我们同时将相同的数据发送到两个不同的 TCP 目的地。

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpStream;
use std::io;
use std::net::SocketAddr;

async fn race(
    data: &[u8],
    addr1: SocketAddr,
    addr2: SocketAddr
) -> io::Result<()> {
    tokio::select! {
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr1).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr2).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        else => {}
    };

    Ok(())
}
```

两个异步表达式不可变地借用 `data` 变量。当其中一项操作成功完成时，另一项操作将被删除。因为我们对 `Ok(_)` 进行模式匹配，所以如果一个表达式失败，另一个表达式会继续执行。

当涉及到每个分支的 `<handler>` 时，`select!` 保证只有一个 `<handler>` 运行。因此，每个 `<handler>` 都可以可变地借用相同的数据。

例如，两个 handler 都修改了 out：

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    let mut out = String::new();

    tokio::spawn(async move {
        // Send values on `tx1` and `tx2`.
    });

    tokio::select! {
        _ = rx1 => {
            out.push_str("rx1 completed");
        }
        _ = rx2 => {
            out.push_str("rx2 completed");
        }
    }

    println!("{}", out);
}
```

## Loops

`select!` 宏通常用在循环中。本节将介绍一些示例，以展示在循环中使用 `select!` 宏。我们首先选择多个渠道：

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx1, mut rx1) = mpsc::channel(128);
    let (tx2, mut rx2) = mpsc::channel(128);
    let (tx3, mut rx3) = mpsc::channel(128);

    loop {
        let msg = tokio::select! {
            Some(msg) = rx1.recv() => msg,
            Some(msg) = rx2.recv() => msg,
            Some(msg) = rx3.recv() => msg,
            else => { break }
        };

        println!("Got {:?}", msg);
    }

    println!("All channels have been closed.");
}
```

此示例针对三个接收通道使用 select。当任意通道上收到一条消息，都将被写入 STDOUT。关闭通道时，`recv()` 返回 None。通过模式匹配，`select!` 宏继续等待其余通道。当所有通道都关闭时，进入 else 分支，并终止循环。

`select!` 宏随机选择分支首先检查就绪情况。当多个通道有待处理值时，将选择一个随机通道来接收。这是为了应对接收循环处理消息的速度比消息推送到通道的速度慢的情况，这意味着通道开始填满。如果 `select!` 没有随机选择一个分支来首先检查，在循环的每次迭代中，都首先检查 `rx1`。如果 `rx1` 总是包含一条新消息，其余通道永远不会被检查。

> 如果 select! 进行检查时，多个通道有待处理的消息，只有一个通道有值弹出。其他所有通道保持不变，并且它们的消息保留在这些通道中，直到下一次循环迭代。消息不会丢失。

### Resuming an async operation

现在我们将展示如何在多次调用 `select!` 中运行异步操作。在此示例中，我们有一个带有类型为 `i32` 的 MPSC 通道和一个异步函数。我们希望运行异步函数指导结束或在通道上收到一个整数为止。

```rust
async fn action() {
    // Some asynchronous logic
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);    
    
    let operation = action();
    tokio::pin!(operation);
    
    loop {
        tokio::select! {
            _ = &mut operation => break,
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    break;
                }
            }
        }
    }
}
```

请注意，没有在 `select!` 宏中调用 `action()`，而是在循环的外部调用它。`action()` 的返回值赋值给了 `operation` 而没有调用 `.await` 。然后我们对 `operation` 使用 `tokio::pin!`。

在 `select!` 循环中，我们没有传递 `operation`，而是传递 `&mut operation`。该 `operation` 变量跟踪正在运行的异步操作。循环的每次迭代都使用相同的操作，而不是发出新的 `action()`。

另一个 `select!` 分支从通道接收消息。如果消息甚至是偶数，则循环结束。否则，再次开始 `select!`。

这是我们第一次使用 `tokio::pin!`。我们还不会介绍 pin 的细节。要注意的是，对引用使用 `.await`，需要将引用的值 pin 住或者实现了 `Unpin`。

如果我们删除 `tokio::pin!` 行并尝试编译，我们会收到以下错误：

```rust
error[E0599]: no method named `poll` found for struct
     `std::pin::Pin<&mut &mut impl std::future::Future>`
     in the current scope
  --> src/main.rs:16:9
   |
16 | /         tokio::select! {
17 | |             _ = &mut operation => break,
18 | |             Some(v) = rx.recv() => {
19 | |                 if v % 2 == 0 {
...  |
22 | |             }
23 | |         }
   | |_________^ method not found in
   |             `std::pin::Pin<&mut &mut impl std::future::Future>`
   |
   = note: the method `poll` exists but the following trait bounds
            were not satisfied:
           `impl std::future::Future: std::marker::Unpin`
           which is required by
           `&mut impl std::future::Future: std::future::Future`
```

尽管我们在[上一章](./async_in_depth.md)介绍了 `Future`，但此错误仍然不太清楚。如果您在尝试对引用调用 `.await` 时遇到有关 Future 未实现的错误时，那么 future 可能需要被 pin 住。

在[标准库](https://doc.rust-lang.org/std/pin/index.html)上阅读有关 [Pin](https://doc.rust-lang.org/std/pin/index.html) 的更多信息。

### Modifying a branch

让我们看一下一个更复杂的循环。我们有：

1. `i32` 值的通道。
2. 在 `i32` 值上执行的异步操作。

我们要实现的逻辑是：

1. 等待通道上的偶数。
2. 使用偶数数字作为输入启动异步操作。
3. 等待操作，但同时监听通道上的更多偶数数字。
4. 如果在现有操作完成之前收到了新的偶数数字，则中止现有操作，然后从新的偶数数字开始。

```rust
async fn action(input: Option<i32>) -> Option<String> {
    // If the input is `None`, return `None`.
    // This could also be written as `let i = input?;`
    let i = match input {
        Some(input) => input,
        None => return None,
    };
    // async logic here
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);
    
    let mut done = false;
    let operation = action(None);
    tokio::pin!(operation);
    
    tokio::spawn(async move {
        let _ = tx.send(1).await;
        let _ = tx.send(3).await;
        let _ = tx.send(2).await;
    });
    
    loop {
        tokio::select! {
            res = &mut operation, if !done => {
                done = true;

                if let Some(v) = res {
                    println!("GOT = {}", v);
                    return;
                }
            }
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    // `.set` is a method on `Pin`.
                    operation.set(action(Some(v)));
                    done = false;
                }
            }
        }
    }
}
```

我们使用与上一个示例相似的策略。异步函数在循环外部调用并赋值给 `operation` 。`operation` 变量被 pin 住。循环针对 `operation` 和通道接收端使用 select。

请注意，`action` 如何将 `Option<i32>` 作为参数。在收到第一个偶数数字之前，我们需要 operation 实例化为某些内容。我们让 `action` 使用 `Option` 作为参数和返回值。如果参数为 `None`，则返回值为 None。第一次循环迭代时， `operation` 立即完成且返回 `None`。

此示例使用了一些新的语法。第一个分支包括 `, if !done`。这是一个分支前提条件。在解释其工作原理之前，让我们看看如果省略前提条件会发生什么。遗漏 `, if !done` 则会导致以下输出：

```rust
thread 'main' panicked at '`async fn` resumed after completion', src/main.rs:1:55
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

此错误发生于在 `operation` 已经完成后再次尝试使用。通常，使用 `.await` 时，要等待的值将被消耗。在此示例中，我们正在等待引用。这意味着 `operation` 完成后仍然存在。

为了避免这种报错，我们必须确保如果 `operation` 已经完成，则第一个分支无效。`done` 变量用于跟踪 `operation` 是否完成。 `select!` 分支可能包括前提条件。在 select! 在分支等待之前，先检查前提条件。如果条件为 `false`，则该分支无效。`done` 变量初始化为 `false` 。`operation` 完成后，将 `done` 设置为 `true`。下一个循环迭代将禁用 `operation` 分支。当从通道接收到偶数消息时，将重置 `operation` 并将 `done` 设置为 `false`。

## Per-task concurrency

`tokio::spawn` 和 `select!` 允许运行并发异步操作。但是，用于运行并发操作的策略有所不同。`tokio::spawn` 函数接收异步操作作为参数并生成一个新的任务来运行。任务是 `Tokio` 运行时调度的对象。 Tokio 独立调度两个不同的任务。它们可能同时在不同的操作系统线程上运行。因此，生成的任务与生成的线程具有相同的限制：无借用。

`select!` 宏在同一个任务上并发运行所有分支，因为 `select!` 宏的所有分支在同一个任务上运行，因此它们不能同时运行。`select!` 宏在单个任务上多路复用异步操作。
