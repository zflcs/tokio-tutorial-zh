# scheduler

## inject

MPMC 队列，当线程的局部队列满了时，将任务插入到这个队列中

对 Synced 和 Shared 进一步封装，提供 close、push、pop 操作

### Synced

侵入式链表，包括链表头尾指针。

实现了 pop 函数

### Shared

与 Synced 一起操作，但是没有将 Synced 直接放在自己的数据结构中，而是将 Synced 作为参数传递到相应的函数中。

在 Synced 的基础上提供了 push 操作

### Pop

实现了对封装了 Synced 的迭代器

### metrics

提供了 len 方法访问队列中的任务数量

### rt_multi_thread

提供了批量插入任务的方法

## defer

提供了一个队列，用于存储 Waker，对于在 .await 时不需要立即唤醒的 Waker，会将其存储在这里。

## current_thread

```rust
pub(crate) struct CurrentThread {
    /// Core scheduler data is acquired by a thread entering `block_on`.
    core: AtomicCell<Core>,

    /// Notifier for waking up other threads to steal the
    /// driver.
    notify: Notify,
}
```

CurrentThread 即为 Tokio 中所述的 Scheduler，但这里记录的 core 指针与 Context 中记录的 core 是同一个，在 block_on 时，会将这个 core 取出来用于构建 Context

```rust
struct Core {
    /// Scheduler run queue
    tasks: VecDeque<Notified>,

    /// Current tick
    tick: u32,

    /// Runtime driver
    ///
    /// The driver is removed before starting to park the thread
    driver: Option<Driver>,

    /// Metrics batch
    metrics: MetricsBatch,

    /// How often to check the global queue
    global_queue_interval: u32,

    /// True if a task panicked without being handled and the runtime is
    /// configured to shutdown on unhandled panic.
    unhandled_panic: bool,
}
```
Core：

- tasks：当前线程的局部队列

调度：

- next_task：取出下一个任务，这里对应了从局部队列还是全局队列中取出任务
- next_local_task：从 tasks（局部队列） 中取出下一个任务
- push_task：将任务放入到 tasks（局部队列中）


```rust
/// Scheduler state shared between threads.
struct Shared {
    /// Remote run queue
    inject: Inject<Arc<Handle>>,

    /// Collection of all active tasks spawned onto this executor.
    owned: OwnedTasks<Arc<Handle>>,

    /// Indicates whether the blocked on thread was woken.
    woken: AtomicBool,

    /// Scheduler configuration options
    config: Config,

    /// Keeps track of various runtime metrics.
    scheduler_metrics: SchedulerMetrics,

    /// This scheduler only has one worker.
    worker_metrics: WorkerMetrics,
}
```

Shared 可以在调度器之间共享

- inject：全局队列
- owned：应该是用于记录可以访问这个 Shared 数据结构的任务

```rust
pub(crate) struct Context {
    /// Scheduler handle
    handle: Arc<Handle>,

    /// Scheduler core, enabling the holder of `Context` to execute the
    /// scheduler.
    core: RefCell<Option<Box<Core>>>,

    /// Deferred tasks, usually ones that called `task::yield_now()`.
    pub(crate) defer: Defer,
}
```

- run_task：执行参数中的闭包中的逻辑
- park：阻塞当前线程，直到 driver 收到了事件，核心的逻辑应该是 driver.park() 这个函数，还包括了一些在 park 前后需要执行的函数
- park_yield：核心逻辑是 driver.park_timeout() 函数

### Handle

```rust
pub(crate) struct Handle {
    /// Scheduler state shared across threads
    shared: Shared,

    /// Resource driver handles
    pub(crate) driver: driver::Handle,

    /// Blocking pool spawner
    pub(crate) blocking_spawner: blocking::Spawner,

    /// Current random number generator seed
    pub(crate) seed_generator: RngSeedGenerator,

    /// User-supplied hooks to invoke for things
    pub(crate) task_hooks: TaskHooks,

    /// If this is a `LocalRuntime`, flags the owning thread ID.
    pub(crate) local_tid: Option<ThreadId>,
}
```

- spawn：创建任务，调用 schedule 函数，如果当前的 context 与当前线程相同，则将任务放入 core 的局部队列中；否则放入到 Shared 的全局队列中
- next_remote_task：从全局队列中取出任务

```rust
/// Used to ensure we always place the `Core` value back into its slot in
/// `CurrentThread`, even if the future panics.
struct CoreGuard<'a> {
    context: scheduler::Context,
    scheduler: &'a CurrentThread,
}
```

- block_on：这个函数定义了 core 怎么运行 future，在 core 不能取出任务时，会通过 park_yield 或 park 检查是否有事件发生，或者已经从 core 中取出了 event_interval 次任务运行了，这个应该算是 CurrentThread 的核心

## MultiThread

MultiThread 没有实质的数据结构。

在这个运行时中，worker 和 core 是两个可以拆分的概念，worker 对应的是线程，而 core 则是对应的任务队列，更贴近调度器（任务队列），worker 的职责是执行，不管 core 是怎么维护任务队列的，当前拿到了哪个 core，就从这个 core 中取出任务。

### Worker

```rust
/// A scheduler worker
pub(super) struct Worker {
    /// Reference to scheduler's handle
    handle: Arc<Handle>,

    /// Index holding this worker's remote state
    index: usize,

    /// Used to hand-off a worker's core to another thread.
    core: AtomicCell<Core>,
}
```

调度器在初始化时创建了固定数量的 worker（线程），每个 worker 包含了 Core 这个数据结构。

```rust
/// Core data
struct Core {
    /// Used to schedule bookkeeping tasks every so often.
    tick: u32,

    /// When a task is scheduled from a worker, it is stored in this slot. The
    /// worker will check this slot for a task **before** checking the run
    /// queue. This effectively results in the **last** scheduled task to be run
    /// next (LIFO). This is an optimization for improving locality which
    /// benefits message passing patterns and helps to reduce latency.
    lifo_slot: Option<Notified>,

    /// When `true`, locally scheduled tasks go to the LIFO slot. When `false`,
    /// they go to the back of the `run_queue`.
    lifo_enabled: bool,

    /// The worker-local run queue.
    run_queue: queue::Local<Arc<Handle>>,

    /// True if the worker is currently searching for more work. Searching
    /// involves attempting to steal from other workers.
    is_searching: bool,

    /// True if the scheduler is being shutdown
    is_shutdown: bool,

    /// True if the scheduler is being traced
    is_traced: bool,

    /// Parker
    ///
    /// Stored in an `Option` as the parker is added / removed to make the
    /// borrow checker happy.
    park: Option<Parker>,

    /// Per-worker runtime stats
    stats: Stats,

    /// How often to check the global queue
    global_queue_interval: u32,

    /// Fast random number generator.
    rand: FastRand,
}
```

这个 Core 与 CurrentThread 运行时中的 Core 定义不同，它增加了与 lifo 相关的字段。

- next_task 函数：这里在函数返回前增加了从全局队列中取出任务到局部队列的操作
- next_local_task 函数：先从 lifo slot 中取，再从局部队列中取
- steal_work 函数：逻辑是从 remote 中随机选取一个目标 worker，然后调用 steal_into 函数；如果失败，则回过头检查全局队列

```rust
/// State shared across all workers
pub(crate) struct Shared {
    /// Per-worker remote state. All other workers have access to this and is
    /// how they communicate between each other.
    remotes: Box<[Remote]>,

    /// Global task queue used for:
    ///  1. Submit work to the scheduler while **not** currently on a worker thread.
    ///  2. Submit work to the scheduler when a worker run queue is saturated
    pub(super) inject: inject::Shared<Arc<Handle>>,

    /// Coordinates idle workers
    idle: Idle,

    /// Collection of all active tasks spawned onto this executor.
    pub(crate) owned: OwnedTasks<Arc<Handle>>,

    /// Data synchronized by the scheduler mutex
    pub(super) synced: Mutex<Synced>,

    /// Cores that have observed the shutdown signal
    ///
    /// The core is **not** placed back in the worker to avoid it from being
    /// stolen by a thread that was spawned as part of `block_in_place`.
    #[allow(clippy::vec_box)] // we're moving an already-boxed value
    shutdown_cores: Mutex<Vec<Box<Core>>>,

    /// The number of cores that have observed the trace signal.
    pub(super) trace_status: TraceStatus,

    /// Scheduler configuration options
    config: Config,

    /// Collects metrics from the runtime.
    pub(super) scheduler_metrics: SchedulerMetrics,

    pub(super) worker_metrics: Box<[WorkerMetrics]>,

    /// Only held to trigger some code on drop. This is used to get internal
    /// runtime metrics that can be useful when doing performance
    /// investigations. This does nothing (empty struct, no drop impl) unless
    /// the `tokio_internal_mt_counters` `cfg` flag is set.
    _counters: Counters,
}
```

- remotes：每个 worker 的 remote 状态
- inject：全局队列，当任务由其他的 worker 创建或者当前 worker 的局部队列饱和时，将任务提交到这里

#### Context

```rust
/// Thread-local context
pub(crate) struct Context {
    /// Worker
    worker: Arc<Worker>,

    /// Core data
    core: RefCell<Option<Box<Core>>>,

    /// Tasks to wake after resource drivers are polled. This is mostly to
    /// handle yielded tasks.
    pub(crate) defer: Defer,
}
```

这里与 CurrentTHread 的定义类似

block_in_place 函数：先将 lifo slot 中的任务放到 core 的局部队列中，由于这个线程会执行阻塞操作，而 lifo slot 中的任务无法被其他的 core 窃取，如果不这样做，会导致 lifo slot 中的任务饥饿。核心逻辑是 `runtime::spawn_blocking(move || run(worker));`

- run 函数：先从 core 中取出下一个任务运行，若无法取出任务，则尝试从其他的 worker 窃取任务，如果窃取不到任务，则调用 park_timeout 和 park 函数检查是否有 IO 事件发生。
- run_task 函数：在调用完 task.run 函数之后，会检查 lifo slot 中的，并且不重置 budget，如果没有 budget，则会将从 lifo 中取出来的任务放到到队列中；如果还剩余 budget，则运行从 lifo slot 中取出的任务。当连续从 lifo slot 中取出任务超过 MAX_LIFO_POLLS_PER_TICK 次数时，会禁用 lifo slot。
- maintenance 函数：调用 park_timeout 检查 IO 或 timer 事件
- park 函数：调用 park_timeout(None)
- park_timeout 函数：根据参数调用 park.park 或 park.park_timeout，之后再调用 defer.wake 函数唤醒一个任务

#### Launch

调用 `runtime::spawn_blocking(move || run(worker));` 来让所有的 worker 开始工作。

#### fn run(worker: Arc<Worker>) {......}

核心逻辑是获取到 context，调用 cx.run() 函数

### Handle

```rust
pub(crate) struct Handle {
    /// Task spawner
    pub(super) shared: worker::Shared,

    /// Resource driver handles
    pub(crate) driver: driver::Handle,

    /// Blocking pool spawner
    pub(crate) blocking_spawner: blocking::Spawner,

    /// Current random number generator seed
    pub(crate) seed_generator: RngSeedGenerator,

    /// User-supplied hooks to invoke for things
    pub(crate) task_hooks: TaskHooks,
}
```

MultiThread 运行时下的调度器：

- spawn 函数：调用 bind_new_task
- bind_new_task 函数：调用 schedule_option_task_without_yield
- schedule_option_task_without_yield 函数：调用 schedule_task
- schedule_task 函数：先判断 task 是否属于当前的 scheduler，如果是，则调用 schedule_local；否则，将其放入全局队列中，并通知
- schedule_local 函数：根据是否 yield 以及 lifo 的使能情况，将任务放到队尾或者 lifo slot 中，并将 lifo slot 中原有的任务放到队尾，并通知
- next_remote_task 函数：从全局队列中取出任务
- push_remote_task 函数：将任务放入全局队列
- notify_parked_local 函数：从 Idle 中找到处于 Search 状态或者 sleeping 状态的 worker id，并调用 unpark.unpark 通知；其他的 notify 函数大同小异

### Queue

- Local：生产者
- Steal：消费者

这两个是对 Arc<Inner> 的封装。

```rust
pub(crate) struct Inner<T: 'static> {
    /// Concurrently updated by many threads.
    ///
    /// Contains two `UnsignedShort` values. The `LSB` byte is the "real" head of
    /// the queue. The `UnsignedShort` in the `MSB` is set by a stealer in process
    /// of stealing values. It represents the first value being stolen in the
    /// batch. The `UnsignedShort` indices are intentionally wider than strictly
    /// required for buffer indexing in order to provide ABA mitigation and make
    /// it possible to distinguish between full and empty buffers.
    ///
    /// When both `UnsignedShort` values are the same, there is no active
    /// stealer.
    ///
    /// Tracking an in-progress stealer prevents a wrapping scenario.
    head: AtomicUnsignedLong,

    /// Only updated by producer thread but read by many threads.
    tail: AtomicUnsignedShort,

    /// Elements
    buffer: Box<[UnsafeCell<MaybeUninit<task::Notified<T>>>; LOCAL_QUEUE_CAPACITY]>,
}
```

最大容量为 LOCAL_QUEUE_CAPACITY。

#### Local

local 函数：创建局部队列，以及 Steal，同时返回 Steal 和 local，两者是相同的
push_back 函数：将一批任务放回到局部队列中
push_back_or_overflow 函数：将局部队列中的一半任务放到全局队列中，之后再把参数中的任务放到原来的局部队列中（调用 push_back_finish）

#### Steal

steal_into 函数：从自己处窃取一半的任务放到 dst 处，实际的操作在 steal_into2 中
steal_into2 函数：返回从自己处窃取的任务数量

pack 和 unpack 函数对队列的 head 以及 stealer 正在操作的位置进行打包和解包

### Park

Parker 和 Unparker 对 Inner 进行封装

```rust
struct Inner {
    /// Avoids entering the park if possible
    state: AtomicUsize,

    /// Used to coordinate access to the driver / `condvar`
    mutex: Mutex<()>,

    /// `Condvar` to block on if the driver is unavailable.
    condvar: Condvar,

    /// Resource (I/O, time, ...) driver
    shared: Arc<Shared>,
}
```

park 函数：调用 park_driver 或 park_condvar
park_driver 函数：调用 driver.park

```rust
/// Shared across multiple Parker handles
struct Shared {
    /// Shared driver. Only one thread at a time can use this
    driver: TryLock<Driver>,
}
```

这里的 driver 是对 TimeDriver 的封装

### Idle

定义了处于 Idle 状态的 worker 线程的相关方法
