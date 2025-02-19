# Task

Task 的引用存在于一下集中封装类型中：

- OwnedTask / LocalOwnedTasks：
- JoinHandle：用于访问 Task 的输出
- Waker：waker 中包含了 Task 引用
- Notified：跟踪任务是否被通知
- Unowned：用于执行阻塞任务，不属于任何 runtime，有两个引用计数，其他引用只有一个引用计数

Task 状态：

- RUNNING：跟踪任务是否正在被 polled 或者 cancelled
- COMPLETE：任务完全结束且被释放掉，不会与 RUNNING 一起设置
- NOTIFIED：跟踪一个通知的对象当前是否存在
- CANCELLED：
- JOIN_INTEREST：如果 Task 存在 JoinHandle 引用，则为 1
- JOIN_WAKER：充当 join handle waker 的控制位

剩余位用于引用计数

## RawTask

```rust
pub(crate) struct RawTask {
    ptr: NonNull<Header>,
}
```

ptr 中存放了关于 Future 的原始指针

这些与其他的 runtime 中定义的 Task 使用的技巧类似，没有继续阅读
