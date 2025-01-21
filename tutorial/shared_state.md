# Shared state

到目前为止，我们有一个可以正常工作的键值服务器。但是，有一个重大缺陷：状态无法跨连接共享。我们将在本文中修复此问题。

## 策略

在 Tokio 中有几种不同的方式来共享状态。

1. 使用互斥锁 (Mutex) 保护共享状态。
2. 产生一个任务来管理状态并使用消息传递对其进行操作。

一般来说，对于简单数据，使用第一种方法；对于需要异步工作（例如 I/O 原语）的数据，使用第二种方法。在本章中，共享状态是 `HashMap`，操作是 `insert` 和 `get`。这两个操作都不是异步的，所以我们将使用 `Mutex`。

下一章将介绍后一种方法。

## 添加 `bytes` 依赖

Mini-Redis 箱不使用 `Vec<u8>`，而是使用 [`bytes`](https://docs.rs/bytes/1/bytes/struct.Bytes.html) 中的 `Bytes`。`Bytes` 的目标是为网络编程提供健壮的字节数组结构。它相对于 `Vec<u8>` 增加的最大功能是浅克隆。换句话说，在 `Bytes` 实例上调用 `clone()` 不会复制底层数据。相反，`Bytes` 实例是某些基础数据的引用计数句柄。`Bytes` 类型大致是 `Arc<Vec<u8>>` 但具有一些附加功能。

要依赖 `bytes`，请将以下内容添加到 `Cargo.toml` 中的 `[dependencies]` 部分：

```rust
bytes = "1"
```

## 初始化 `HashMap`

`HashMap` 将在许多任务和可能的许多线程之间共享。为了支持这一点，它被包装在 `Arc<Mutex<_>>` 中。

首先，为了方便起见，在 `use` 语句之后添加以下类型别名。

```rust
use bytes::Bytes;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

type Db = Arc<Mutex<HashMap<String, Bytes>>>;
```

然后，更新 `main` 函数以初始化 `HashMap` 并将 `Arc` 句柄传递给 `process` 函数。使用 `Arc` 允许许多任务同时引用 `HashMap`，这些任务可能在许多线程上运行。在整个 Tokio 中，术语 handle 用于引用提供对某些共享状态的访问的值。

```rust
use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    println!("Listening");

    let db = Arc::new(Mutex::new(HashMap::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Clone the handle to the hash map.
        let db = db.clone();

        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}
```

### 关于使用 `std::sync::Mutex` 和 `tokio::sync::Mutex`

请注意，使用 `std::sync::Mutex` 而不是 `tokio::sync::Mutex` 来保护 `HashMap`。一个常见的错误是在异步代码中无条件地使用 `tokio::sync::Mutex`。异步 `Mutex` 是能够跨越 `.await` 调用的 `Mutex`。

同步 `Mutex` 在等待获取锁时会阻塞当前线程。反过来，这将阻止其他任务的处理。但是，切换到 `tokio::sync::Mutex` 通常没有帮助，因为异步 `Mutex` 在内部使用同步 `Mutex`。

根据经验，在异步代码中使用同步 `Mutex` 是只要竞争保持在低水平并且在调用 `.await` 之间不持有锁就可以了。

## 更新 `process()`

process 函数不再初始化 `HashMap`。相反，它将 `HashMap` 的共享句柄作为参数。它在使用 `HashMap` 之前需要先持有锁。请记住，`HashMap` 的值的类型现在是 `Bytes` （我们可以廉价地克隆），因此也需要更改。

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream, db: Db) {
    use mini_redis::Command::{self, Get, Set};

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                let mut db = db.lock().unwrap();
                db.insert(cmd.key().to_string(), cmd.value().clone());
                Frame::Simple("OK".to_string())
            }           
            Get(cmd) => {
                let db = db.lock().unwrap();
                if let Some(value) = db.get(cmd.key()) {
                    Frame::Bulk(value.clone())
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

## 在 `.await` 上持有 `MutexGuard`

您可能会编写如下所示的代码：

```rust
use std::sync::{Mutex, MutexGuard};

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
```

当您尝试生成调用此函数的内容时，您将遇到以下错误消息：

```shell
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

发生这种情况是因为 `std::sync::MutexGuard` 类型没有实现 `Send`。这意味着您无法将互斥锁发送到另一个线程，并且会发生错误，因为 Tokio 运行时可以在每个 `.await` 处在线程之间移动任务。为了避免这种情况，您应该重构代码，使互斥锁的析构函数在 `.await` 之前运行。

```rust
// This works!
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    } // lock goes out of scope here

    do_something_async().await;
}
```

请注意，这不起作用：

```rust
use std::sync::{Mutex, MutexGuard};

// This fails too.
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);

    do_something_async().await;
}
```

这是因为编译器当前计算 future 是否为 `Send` 时，仅基于作用域信息。编译器有望更新为支持显式删除 future，但现在，您必须显式地使用作用域。

请注意，这里讨论的错误也在 [Send bound section from the spawning chapter](https://tokio.rs/tokio/tutorial/spawning#send-bound) 中进行了讨论。

您不应该尝试通过以不需要 `Send` 方式生成任务来规避此问题，因为如果 Tokio 在某个 `.await` 时间挂起您的任务，当任务持有锁时，可能会安排其他一些任务在同一线程上运行，并且该其他任务也可能尝试获取该互斥锁，这将导致死锁，因为等待占用的互斥锁的任务将防止持有互斥锁的任务释放互斥锁。

请记住，某些 Mutex crate 为其 MutexGuard 实现了 `Send`。在这种情况下，即使您在 `.await` 上持有 MutexGuard，也不会出现编译器错误。代码可以编译，但死锁！

我们将在下面讨论一些避免这些问题的方法：

### 重构您的代码，使其不跨 `.await` 持有锁

处理互斥锁的最安全方法是将其包装在一个结构中，并仅将互斥锁锁定在该结构上的非异步方法内。

```rust
use std::sync::Mutex;

struct CanIncrement {
    mutex: Mutex<i32>,
}
impl CanIncrement {
    // This function is not marked async.
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}

async fn increment_and_do_stuff(can_incr: &CanIncrement) {
    can_incr.increment();
    do_something_async().await;
}
```

此模式保证您不会遇到 `Send` 错误，因为 Mutex guard 不会出现在异步函数中的任何位置。当使用为 `MutexGuard` 实现 `Send` 的 crate 时，它还可以保护您免受死锁的影响。

您可以在此[博文](https://draft.ryhl.io/blog/shared-mutable-state/)中找到更详细的示例。

### 生成一个任务来管理状态并使用消息传递来对其进行操作

这是本章开头提到的第二种方法，当共享资源是 I/O 资源时经常使用。有关详细信息，请参阅下一章。

### 使用 Tokio 的异步互斥体

也可以使用 Tokio 提供的 `tokio::sync::Mutex` 类型。 Tokio Mutex 的主要功能是它可以在 `.await` 中保持，不会出现任何问题。也就是说，异步 Mutex 比普通 Mutex 更昂贵，并且通常最好使用其他两种方法之一。

```rust
use tokio::sync::Mutex; // note! This uses the Tokio mutex

// This compiles!
// (but restructuring the code would be better in this case)
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
```

## 任务、线程和竞争

当竞争最少时，使用阻塞 Mutex 来保护 short 关键部分是一种可接受的策略。当锁被争用时，执行任务的线程必须阻塞并等待 Mutex。这不仅会阻塞当前任务，还会阻塞当前线程上调度的所有其他任务。

默认情况下，Tokio 运行时使用多线程调度程序。任务被调度到运行时管理的任意数量的线程上。如果大量任务被安排执行并且它们都需要访问 Mutex，就会导致竞争。另一方面，如果使用 [`current_thread`](https://docs.rs/tokio/1/tokio/runtime/index.html#current-thread-scheduler) 运行时，那么互斥锁将永远不会被争用。

> [`current_thread`](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread) 运行时是一个轻量级的单线程运行时。当仅生成一些任务并打开少量套接字时，这是一个不错的选择。例如，当在异步客户端库之上提供同步 API 时，此选项效果很好。

如果同步 Mutex 上的竞争成为问题，最好的解决办法很少是切换到 Tokio 互斥锁。相反，需要考虑的选项是：

1. 让专用任务管理状态并使用消息传递。
2. 对 Mutex 进行分片。
3. 重构代码以避免互斥。

### Mutex 分片

在我们的例子中，由于每个键都是独立的，因此 Mutex 分片效果很好。为此，我们将引入 `N` 不同的实例，而不是使用单个 `Mutex<HashMap<_, _>>` 实例。

```rust
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;

fn new_sharded_db(num_shards: usize) -> ShardedDb {
    let mut db = Vec::with_capacity(num_shards);
    for _ in 0..num_shards {
        db.push(Mutex::new(HashMap::new()));
    }
    Arc::new(db)
}
```

然后，找到任何给定键的单元格就变成了一个两步过程。首先，键用于识别它属于哪个分片。然后，在 `HashMap` 中查找键。

```rust
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

上面概述的简单实现需要使用固定数量的分片，并且一旦创建分片映射，分片的数量就无法更改。

[dashmap](https://docs.rs/dashmap) crate 提供了更复杂的分片 hash map 的实现。您还可以看看诸如 [leapfrog](https://docs.rs/leapfrog) 和 [flurry](https://docs.rs/flurry) 之类的并发 hash table 实现，后者是 Java 的端口 `ConcurrentHashMap` 数据结构。

在开始使用任何这些 crate 之前，请确保构建代码，以便不能在 `.await` 中持有`MutexGuard`。如果不这样做，您将出现编译器错误（在  non-Send guards 的情况下），或者您的代码将死锁（在 Send guards 的情况下）。请参阅此[博文](https://draft.ryhl.io/blog/shared-mutable-state/)中的完整示例和更多上下文。
