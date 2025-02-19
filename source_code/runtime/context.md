# Context

维护了一下数据结构：

- thread_id：线程 id
- current：当前线程运行时的调度器的 handle
- scheduler：当前调度器内部的 context 的 handle
- current_task_id：当前运行的任务 id
- runtime：跟踪当前线程是否正在运行时的上下文中，当被设置为 entered 时（我认为是 runtime 有一个自己的任务/上下文，这个任务是由其他的runtime 来调度的，是一个嵌套的关系），当前运行时调度器的 handler 可能不指向当前的 runtime，可能是指向其他 runtime 的
- budge：任务运行的预算

## current

SetCurrentGuard 应该是只用于进入 runtime 自己的任务/上下文时才会使用到，在进入和退出时要保存和恢复 scheduler 的 handle。

## Scoped

set 方法接受一个闭包，以及要设置的 value 作为参数，在执行闭包期间，Scoped 中的值被暂时替换成要设置的 value，等闭包结束后，自动换回原来的 value。

with 方法将 Scoped 中 value 作为参数传递给闭包来执行

## BlockingRegionGuard

需要先获取到这个结构，才可以将执行阻塞操作的任务绑定到当前线程上

## runtime

EnterRuntime：表示当前所处的上下文，如果为 enter，则表示进入了 runtime 自己的任务/上下文中；否则，表示当前既不处于 runtime 自己的任务/上下文中，也不处于执行阻塞操作的上下文中

EnterRuntimeGuard：

```rust
#[must_use]
pub(crate) struct EnterRuntimeGuard {
    /// Tracks that the current thread has entered a blocking function call.
    pub(crate) blocking: BlockingRegionGuard,

    #[allow(dead_code)] // Only tracking the guard.
    pub(crate) handle: SetCurrentGuard,

    // Tracks the previous random number generator seed
    old_seed: RngSeed,
}
```

封装了对进入 runtime 自己的任务/上下文的一些保护结构
