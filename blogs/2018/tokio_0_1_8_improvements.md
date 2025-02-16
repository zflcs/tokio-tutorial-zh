# Tokio 0.1.8 with many incremental improvements

它花费的时间比我最初希望的要长一些（就像往常一样），但是新的 Tokio 版本已经发布了。除其他功能外，此版本还包含一组新的 API，允许从异步上下文执行文件系统操作、并发改进、计时器改进等（包括错误修复，因此请务必更新！）。

距离上一篇文章已经过去了一段时间。虽然没有发布任何重大功能，但这并不意味着我们一直闲着。在过去的几个月里，我们发布了新的 crate，其中包含许多渐进式改进。这些改进很多都是由社区贡献的，所以我认为有必要稍微强调一下。

## Filesystem APIs

`tokio-fs` 的初始版本更像是一个存根，而不是一个完整的实现。它只包括基本的文件系统操作。

最新版本包含大多数文件系统 API 的[非阻塞版本](https://docs.rs/tokio/0.1.8/tokio/fs/index.html)。这项令人惊叹的工作主要由 [@griff](https://github.com/griff) 在一个[史诗级 PR](https://github.com/tokio-rs/tokio/pull/494) 中和 [@lnicola](https://github.com/lnicola) 在一系列较小的 PR 中贡献，但许多其他人也参与其中，帮助审查和改进了 crate。

感谢：[@dekellum](https://github.com/dekellum)、[@matsadler](https://github.com/matsadler)、[@debris](https://github.com/debris)、[@mati865](https://github.com/mati865)、[@lovebug356](https://github.com/lovebug356)、[@bryanburgers](https://github.com/bryanburgers)、[@shepmaster](https://github.com/shepmaster)。

## Concurrency improvements

在过去的几个月里，[@stjepang](https://github.com/stjepang) 一直在努力改进 Tokio 的并发相关部分。一些亮点：

- [#459](https://github.com/tokio-rs/tokio/issues/459) - 修复线程唤醒中的竞争问题
- [#470](https://github.com/tokio-rs/tokio/issues/470) - 改进 worker spinning
- [#517](https://github.com/tokio-rs/tokio/issues/517) - 提高 reactor 中使用的 RW Lock 的可扩展性。
- [#534](https://github.com/tokio-rs/tokio/issues/534) - 改进工作窃取运行时的窃取部分。

当他来参加 Rustconf 时，我们也进行了愉快的交谈，我对他即将开展的工作感到很兴奋。

当然，感谢 [Crossbeam](https://github.com/crossbeam-rs/) 所做的一切工作。Tokio 很大程度上依赖它。

## `current_thread::Runtime`

自从 [@vorner](https://github.com/vorner) 和 [@kpp](https://github.com/kpp) 首次引入以来，`current_thread::Runtime` 也得到了许多渐进式改进。

[@sdroege](https://github.com/sdroege) 添加了 `Handle`，允许从其他线程在运行时生成任务 ([#340](https://github.com/tokio-rs/tokio/issues/340))。这是使用通道将任务发送到运行时线程（与 `tokio-core` 使用的策略类似）实现的。

[@jonhoo](https://github.com/jonhoo) 实现了 `block_on_all` 函数 ([#477](https://github.com/tokio-rs/tokio/issues/477))，并修复了跟踪活跃 future 数量和协调关闭的错误 ([#478])

### Timer improvements

`tokio::timer` 确实获得了一个新功能：[`DelayQueue`](https://docs.rs/tokio-timer/0.2.6/tokio_timer/struct.DelayQueue.html)。此类型允许用户存储在一段时间后返回的值。这对于支持更复杂的时间相关情况很有用。

让我们以缓存为例。缓存的目标是在一定时间内保存与某个键相关联的值。时间过后，该值将被删除。一直可以用 [`tokio::timer::Delay`](https://docs.rs/tokio-timer/0.2.6/tokio_timer/struct.Delay.html) 来实现这一点，但有点困难。当缓存有很多条目时，必须扫描所有条目以检查是否需要删除它们。

使用 [`DelayQueue`](https://docs.rs/tokio-timer/0.2.6/tokio_timer/struct.DelayQueue.html) 之后，实现变得更加高效：

```rust
#[macro_use]
extern crate futures;
extern crate tokio;
use tokio::timer::{delay_queue, DelayQueue, Error};
use futures::{Async, Poll, Stream};
use std::collections::HashMap;
use std::time::Duration;

struct Cache {
    entries: HashMap<CacheKey, (Value, delay_queue::Key)>,
    expirations: DelayQueue<CacheKey>,
}

const TTL_SECS: u64 = 30;

impl Cache {
    fn insert(&mut self, key: CacheKey, value: Value) {
        let delay = self.expirations
            .insert(key.clone(), Duration::from_secs(TTL_SECS));

        self.entries.insert(key, (value, delay));
    }

    fn get(&self, key: &CacheKey) -> Option<&Value> {
        self.entries.get(key)
            .map(|&(ref v, _)| v)
    }

    fn remove(&mut self, key: &CacheKey) {
        if let Some((_, cache_key)) = self.entries.remove(key) {
            self.expirations.remove(&cache_key);
        }
    }

    fn poll_purge(&mut self) -> Poll<(), Error> {
        while let Some(entry) = try_ready!(self.expirations.poll()) {
            self.entries.remove(entry.get_ref());
        }

        Ok(Async::Ready(()))
    }
}
```

## Many other small improvements

除了上面列出的内容之外，Tokio 还在大多数包中得到了许多小改进和错误修复。这些都是由我们出色的社区提供的。我希望随着时间的推移，越来越多的人会加入到 Tokio 的建设中，并帮助它继续发展。

因此，非常感谢你们所有人迄今为止对 Tokio 做出的[贡献](https://github.com/tokio-rs/tokio/graphs/contributors)。
