# New Timer implementation

为了结束美好的一周，Tokio 发布了[新版本](https://crates.io/crates/tokio/0.1.5)。此版本包含一个全新的计时器实现。

## Timers

有时（通常），人们希望执行与时间相关的代码。也许某个函数需要在特定时刻运行。也许读取需要限制在固定的持续时间内。为了处理时间，人们需要访问计时器！

## Some history

`tokio-timer` crate 已经存在一段时间了。它最初是使用 [hashed timer wheel](http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf)（pdf warning）构建的。它的粒度为 100 毫秒，因此任何分辨率小于 100 毫秒的超时设置都会被四舍五入。通常，在基于网络的应用程序环境中，这是可以的。超时通常至少为 30 秒，并且不需要高精度。

但是，有些情况下 100 毫秒太粗略了。此外，Tokio 计时器的原始实现存在许多恼人的错误，并且由于其采用的实现策略，无法很好地处理边缘情况。

## A new beginning

该计时器已从头开始重写并发布为 [`tokio-timer` 0.2](https://crates.io/crates/tokio-timer/0.2.0)。在大多数情况下，API 非常相似，但实现却完全不同。

它不是仅仅使用单个 hashed timer wheel 实现，而是使用分层方法（也在上面链接的论文中描述）。

计时器使用六个独立级别。每个级别都是一个包含 64 个槽的 hashed wheel。最低级别的槽代表一毫秒。下一级别代表 64 毫秒（1 x 64 个时隙），依此类推。因此，每一级上的时隙覆盖的时间量与下面整个级别的时间量相同。

当设置超时时，如果距离当前时刻在 64 毫秒以内，则进入最低级别。如果超时在 64 毫秒和 4,096 毫秒之间，则进入第二级别，依此类推。

随着时间的流逝，最低级别的超时将被触发。一旦到达最低级别的末尾，下一级别的所有超时都将从该级别移除并移至最低级别。

使用此策略，所有计时器操作（创建超时、取消超时、触发超时）都是恒定的。即使有大量未完成的超时，也能实现非常好的性能。

## A quick look at the API.

如上所述，API 实际上并没有改变。主要有三种类型：

- [`Delay`](https://docs.rs/tokio/0.1/tokio/timer/struct.Delay.html)：在设定的时间点完成的 future。
- [`Timeout`](https://docs.rs/tokio/0.1/tokio/timer/struct.Timeout.html)：修饰 future，确保它在达到超时之前完成。
- [`Interval`](https://docs.rs/tokio/0.1/tokio/timer/struct.Interval.html)：以固定间隔产生值的流。

举一个简单的例子：

```rust
use tokio::prelude::*;
use tokio::timer::Delay;

use std::time::{Duration, Instant};

fn main() {
    let when = Instant::now() + Duration::from_millis(100);
    let task = Delay::new(when)
        .and_then(|_| {
            println!("Hello world!");
            Ok(())
        })
        .map_err(|e| panic!("delay errored; err={:?}", e));

    tokio::run(task);
}
```

上述示例创建了一个新的 `Delay` 实例，它将在 100 毫秒后完成。`new` 函数接受一个 `Instant`，因此我们计算从现在起 100 毫秒后的时间。

一旦达到该时刻，`Delay` future 就会完成，从而导致 `and_then` 块被执行。

此版本附带一个简短的指南，解释如何使用计时器和 [API 文档](https://docs.rs/tokio/0.1/tokio/timer/index.html)。

## Integrated in the Runtime

使用计时器 API 需要运行计时器实例。Tokio [runtime](https://docs.rs/tokio/0.1/tokio/runtime/index.html) 会为您完成所有设置。

当使用 `tokio::ru`n 或直接调用 `Runtime::new` 启动运行时，会启动一个线程池。每个工作线程都会获得一个计时器实例。因此，这意味着如果运行时启动 4 个工作线程，那么就会有 4 个计时器实例，每个线程一个。这样做允许使用计时器而无需支付同步成本，因为计时器将与使用各种计时器类型（延迟、超时、间隔）的代码位于同一线程上。
