# LocalRuntime

```rust
#[derive(Debug)]
#[cfg_attr(docsrs, doc(cfg(tokio_unstable)))]
pub struct LocalRuntime {
    /// Task scheduler
    scheduler: LocalRuntimeScheduler,

    /// Handle to runtime, also contains driver handles
    handle: Handle,

    /// Blocking pool handle, used to signal shutdown
    blocking_pool: BlockingPool,

    /// Marker used to make this !Send and !Sync.
    _phantom: PhantomData<*mut u8>,
}

#[derive(Debug)]
pub(crate) enum LocalRuntimeScheduler {
    /// Execute all tasks on the current-thread.
    CurrentThread(CurrentThread),
}
```

可以不通过 `LocalSet` 来执行没有实现 `Send + Sync` 的任务，这个运行时不能在线程之间迁移，并且在这个运行时内不能使用 `LocalSet`。实际上它是对 `current_thread` 的封装。