# task

这个模块主要提供的是关于 task 的 API，与 task 相关的数据结构在 runtime 中定义

shutdown 函数需要在下一次 await 时才起作用，或者已经处于让权状态。

使用 spawn_blocking 创建的任务不可以被 shutdown，因为这些任务不是 async 的，即使进行了操作，任务仍将继续运行。但如果任务尚未开始运行，则会阻止这个任务开始。

提供了两个 API 用于执行阻塞操作：spawn_blocking 和 block_in_place。

在 async 函数中调用 non-async 函数，需要保证 non-async 函数不会执行阻塞操作。

spawn_blocking：创建阻塞函数，由专用的线程池来执行

block_in_place：会将当前 worker 线程上的其他任务迁移到其他的 worker 线程上，当前的 worker 线程转化为 blocking 线程

## yield

`context::defer(cx.waker());` 处理 waker，当使用了其他的运行时时，会直接调用 `waker.wake_by_ref();` 函数，直接唤醒任务，而在 tokio 的运行时中，则会调用 `scheduler.defer(waker);` 函数，将 waker 放入到一个队列中，不会直接唤醒

## task-local

提供了一个当前线程可以访问的变量，并且提供了一些与 Future 相关的实现

## spawn

创建一个异步任务，并返回 JoinHandle，即使不对 JoinHandle 进行 .await，创建的任务也会在后台执行。可能在当前线程上也可能处于其他线程。

当 Future 的大小超过 BOX_FUTURE_THRESHOLD 时，会将 Future 创建在堆上，根据不同的 runtime，行为不同：

- current_thread：如果处于其他的运行时上下文中，会直接放入到 remote_run_queue 中
- multi_thread：会先在局部队列中调度，不成功再放入到 remote_run_queue 中

## local

