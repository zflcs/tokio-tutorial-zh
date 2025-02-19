# Time

需要重复调用 park 或者 park_timeout 才可以驱动，分辨率为 1ms。

## Driver

```rust
pub(crate) struct Driver {
    /// Parker to delegate to.
    park: IoStack,
}
```

```rust
/// Timer state shared between `Driver`, `Handle`, and `Registration`.
struct Inner {
    /// The earliest time at which we promise to wake up without unparking.
    next_wake: AtomicOptionNonZeroU64,

    /// Sharded Timer wheels.
    wheels: RwLock<ShardedWheel>,

    /// Number of entries in the sharded timer wheels.
    wheels_len: u32,

    /// True if the driver is being shutdown.
    pub(super) is_shutdown: AtomicBool,

    // When `true`, a call to `park_timeout` should immediately return and time
    // should not advance. One reason for this to be `true` is if the task
    // passed to `Runtime::block_on` called `task::yield_now()`.
    //
    // While it may look racy, it only has any effect when the clock is paused
    // and pausing the clock is restricted to a single-threaded runtime.
    #[cfg(feature = "test-util")]
    did_wake: AtomicBool,
}
```

park_internal 函数：调用 IoStack 的 park_timeout 函数
