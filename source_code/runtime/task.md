# Task

与 task 相关的数据结构以及方法对应的源码在 tokio/src/runtime/task/ 中，我对它的个人理解记录在 [task.km](./task.km) 和 [task_state](./task_state.md) 中，Tokio 针对任务的状态进行了详细的设计，扩展了 Rust future 运行阶段的 Pending 和 Ready 状态，手动维护任务的引用计数。

与任务相关的调度器、Future 以及结果的详细设计在 `Core<T: Future, S>` 这个结构和它所提供的方法中，这里更多的是提供一些与数据结构相关的功能。与任务的状态、任务在队列中位置等相关的设计在 `Header` 中体现。而 `Trailer` 则是提供 waker、回调函数相关的，这些是与 waker、reactor 相关。

在创建任务时，会直接创建三个引用，OwnedTask 引用交给调度器本身，所有处于活跃状态（没有结束）的任务都有调度器持有这个引用，但不参与任务调度。JoinHandle 引用类似于 std::Thread 类似，而 Notified 引用则是放到任务队列中的，用于调度。Tokio 手动维护任务的引用计数。

## join

提供了 JoinHandle 这个结构，与 std::thread::JoinHandle 类似。

## list

侵入任务链表被封装为 List<S>，实质是 `ShardedList<Task<S>, <Task<S> as Link>::Target>`：

```rust
pub(crate) struct ShardedList<L, T> {
    lists: Box<[Mutex<LinkedList<L, T>>]>,
    added: MetricAtomicU64,
    count: MetricAtomicUsize,
    shard_mask: usize,
}
```

ShardedList 提供了以下方法：
  
- new：创建一个容量为 size 的侵入式链表
- shard_inner：获取下标为 id 的指定的链表 
- pop_back：先根据参数提供的 shard_id 获取到对应链表，再从链表中探出最后一个元素，并且 count-1
- remove：从链表中删除指定节点。先从节点获取到目标链表的 id，再根据 id 获取到对应的链表，再来删除节点
- lock_shard：获取到参数对应的链表的 lock_guard，从而可以对其进行写操作

其余的用于 runtime 中的链表实现的描述在 [list.km](./list.km) 中。
