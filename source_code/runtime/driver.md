# Driver

核心的结构为 Handle，在调度器的 Handle 数据结构中，存在着一个与 IO 资源相关的 `driver: driver::Handle` 字段。

`driver::Handle` 结构如下：

```rust
pub(crate) struct Handle {
    /// IO driver handle
    pub(crate) io: IoHandle,

    /// Signal driver handle
    #[cfg_attr(any(not(unix), loom), allow(dead_code))]
    pub(crate) signal: SignalHandle,

    /// Time driver handle
    pub(crate) time: TimeHandle,

    /// Source of `Instant::now()`
    #[cfg_attr(not(all(feature = "time", feature = "test-util")), allow(dead_code))]
    pub(crate) clock: Clock,
}
```

详细的信息见 [driver.km](driver.km)