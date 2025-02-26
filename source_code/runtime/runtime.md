# runtime

提供三类服务：

1. schedule
2. IO
3. Timer

共享 runtime 的几种方式

- 使用 Arc
- 使用 Handle：使用 Arc 或者 Handle 可以在多个线程上共享运行时，两者的区别是 runtime 的关闭。调用 shutdown_background 或 shutdown_timeout 需要对 runtime 进行互斥访问，使用 Arc 时，可以通过 try_unwrap。
- 进入 runtime 的上下文中：通过 Runtime::enter 或者 Handle::enter。当进入 runtime 时，tokio::spawn 将会使用当前所在上下文的运行时。

详细的信息见 [runtime.km](./runtime.km)

## IO 和 timer

周期性检查 IO 资源和 timer 是否就绪，唤醒相关的任务，并进行调度

## scheudle

### current_thread

维护了两个 FIFO，一个全局队列和一个局部队列，选择下一个任务时，先从局部队列中取出任务，若局部队列为空，或者已经从局部队列中取出 global_queue_interval 次任务后，从全局队列中取出下一个任务

当没有任务或者已经调度了 event_interval 次任务后，runtime 将会检查是否出现了 IO 或者 timer 事件。

当一个任务是由一个正在运行的任务唤醒时，它会直接放在局部队列中，否则放入全局队列。

没有使用 lifo slot 优化。

具体的描述在 [current_thread.km](./current_thread.km) 和 [scheduler](./scheduler.md) 中（两个文件中有重复的地方，使用脑图会更加清楚的帮助理解）。但目前还不理解与 IO、Timer 这些之间如何进行交互，只知道是通过各自的 driver 的 park 方法以及 notify 来进行。

### multi_thread

初始化时创建固定数量的 worker 线程，维护一个全局队列和与 worker 线程数量相同的局部队列。每个局部队列最多支持 256 个任务。如果局部队列中的任务数量超过 256，则会将其中一半迁移至全局队列中。

选择下一个任务的策略与 current_thread 相同。如果在构建 runtime 时没有明确设置 global_queue_interval，则使用启发式方法（每 10ms 检查一次全局队列，基于 worker_mean_poll_time），动态计算该值。

当局部队列和全局队列都为空时，会尝试从其他的局部队列窃取任务（将一半的任务从该局部队列迁移至自己的局部队列）。

IO 和 timer 事件与 current_thread 相同。

使用了 lifo slot 优化，当一个任务唤醒了其他任务时，被唤醒的任务被添加到 lifo slot 中，而不是放入队列中，若 lifo slot 中已经存在任务啊，则原来的任务会被放入到线程的局部队列中，新的任务取代原来的任务被放置到 lifo slot 中。优先调度 lifo slot 中的任务。使用 lifo slot 时，不会重置 budget，当 worker 线程连续 3 次使用 lifo slot 时，lifo slot 会被暂时禁用（disable_lifo_slot），直到下一次调度的任务不是来自 lifo slot。其他的线程不能从 lifo slot 中窃取任务。

当任务是被除了 worker 线程之外的其他线程唤醒时，被唤醒的任务会被放置在全局队列中。

更详细的描述见 [multi_thread.km](./multi_thread.km)。

截止目前为止，tokio runtime 中关于任务控制块，任务调度的大部分已经理解了，但是与外部事件相关的 park、unpark 等操作还没有完全理解，以及 worker 之间、外部事件与 worker 之间如何 notify 还未开始。