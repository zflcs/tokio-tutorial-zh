# An RFC for a Tokio revamp

大家好，Tokio 社区！

Carl、Alex 和我一直在努力开发简化、精简和集中 Tokio 项目的方法。作为这项工作的一部分，我们编写了有史以来第一个 Tokio [RFC](https://github.com/carllerche/tokio-rfcs/pull/2)！

以下是所提建议的简要概述。

- 在 tokio-core 中添加一个默认自动管理的全局事件循环。此更改消除了在绝大多数情况下设置和管理自己的事件循环的需要。
  - 此外，通过将 `Handle` 设置为 `Send` 和 `Sync` 并弃用 `Remote`，消除了 `tokio-core` 中 `Handle` 和 `Remote` 之间的区别。这样，即使使用自定义事件循环也会变得更简单。
- 将所有任务执行功能与 Tokio 分离，而是通过标准 futures 组件提供。与事件循环一样，提供一个满足大多数用例的默认全局线程池，无需任何手动设置。
  - 此外，在本地线程运行任务时（对于非 `Send` futures），提供更多万无一失的 API，有助于避免丢失唤醒。
- 在新的 `tokio` crate 中提供上述更改，它是当今 `tokio-core` 的精简版，并且最终可能会重新导出 `tokio-io` 的内容。`tokio-core` crate 已弃用，但仍可用于向后兼容。从长远来看，大多数用户只需要依赖 `tokio` 即可使用 Tokio stack。
- 文档主要关注 `tokio`，而不是 `tokio-proto`。未来将提供更广泛的食谱风格示例和一般指南，以及更深入的操作指南。

总而言之，这些变化与 [async/await](https://internals.rust-lang.org/t/help-test-async-await-generators-coroutines/5835) 一起，应该会在很大程度上使 Tokio 成为一个对新手友好的库。请查看 [RFC](https://github.com/carllerche/tokio-rfcs/pull/2) 并留下您的反馈！

一旦我们就 RFC 达成共识，我们计划组建一个 impl period 工作组，主要关注文档和示例。从那里开始，我们将与 Hyper 团队合作，找出这个故事的下一章。敬请期待！

—— Aaron Turon
