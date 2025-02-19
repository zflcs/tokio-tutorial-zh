# blocking

## BlockingTask

对 non-async 闭包的封装，实现了 Future trait，因此可以使用 .await，在运行闭包之前，会强制取消 budge 设置

## shutdown channel

oneshot channel，每个 worker 持有 oneshot channel 的 Sender 端，所有的 Sender 关闭后，Receiver 会收到通知。rx 端等待 tx 端的消息，可能会阻塞，因此需要进入 blocking_region

## BlockingSchedule

没有做实质性的内容，只是实现了 task::Schedule 这个 trait，作为一个标记

## BlockingPool

```rust
pub(crate) struct BlockingPool {
    spawner: Spawner,
    shutdown_rx: shutdown::Receiver,
}
```

spawner 中的实际内容为 Inner：

```rust
struct Inner {
    /// State shared between worker threads.
    shared: Mutex<Shared>,

    /// Pool threads wait on this.
    condvar: Condvar,

    /// Spawned threads use this name.
    thread_name: ThreadNameFn,

    /// Spawned thread stack size.
    stack_size: Option<usize>,

    /// Call after a thread starts.
    after_start: Option<Callback>,

    /// Call before a thread stops.
    before_stop: Option<Callback>,

    // Maximum number of threads.
    thread_cap: usize,

    // Customizable wait timeout.
    keep_alive: Duration,

    // Metrics about the pool.
    metrics: SpawnerMetrics,
}
```

shared 由以下内容构成

```rust
struct Shared {
    queue: VecDeque<Task>,
    num_notify: u32,
    shutdown: bool,
    shutdown_tx: Option<shutdown::Sender>,
    /// Prior to shutdown, we clean up `JoinHandles` by having each timed-out
    /// thread join on the previous timed-out thread. This is not strictly
    /// necessary but helps avoid Valgrind false positives, see
    /// <https://github.com/tokio-rs/tokio/commit/646fbae76535e397ef79dbcaacb945d4c829f666>
    /// for more information.
    last_exiting_thread: Option<thread::JoinHandle<()>>,
    /// This holds the `JoinHandles` for all running threads; on shutdown, the thread
    /// calling shutdown handles joining on these.
    worker_threads: HashMap<usize, thread::JoinHandle<()>>,
    /// This is a counter used to iterate `worker_threads` in a consistent order (for loom's
    /// benefit).
    worker_thread_index: usize,
}
```

执行阻塞操作的任务的定义：

```rust
pub(crate) struct Task {
    task: task::UnownedTask<BlockingSchedule>,
    mandatory: Mandatory,
}
```

spawn 出来的执行阻塞操作的任务不会直接与 spawn 的线程进行绑定，而是将任务放在一个队列中，由线程从队列中取出后来运行，spawn 出来的线程没有明确的标识一定是用于执行阻塞操作的任务，而是确保了线程数量的一直，多出来一个单独的线程来执行阻塞操作，其余的任务仍然在其他的线程上进行多路复用。