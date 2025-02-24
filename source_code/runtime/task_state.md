# TaskState

对 Tokio 中的任务状态的理解。

```rust
/// The task is currently being run.
const RUNNING: usize = 0b0001;

/// The task is complete.
///
/// Once this bit is set, it is never unset.
const COMPLETE: usize = 0b0010;

/// Extracts the task's lifecycle value from the state.
const LIFECYCLE_MASK: usize = 0b11;

/// Flag tracking if the task has been pushed into a run queue.
const NOTIFIED: usize = 0b100;

/// The join handle is still around.
const JOIN_INTEREST: usize = 0b1_000;

/// A join handle waker has been set.
const JOIN_WAKER: usize = 0b10_000;

/// The task has been forcibly cancelled.
const CANCELLED: usize = 0b100_000;

/// All bits.
const STATE_MASK: usize = LIFECYCLE_MASK | NOTIFIED | JOIN_INTEREST | JOIN_WAKER | CANCELLED;

/// Bits used by the ref count portion of the state.
const REF_COUNT_MASK: usize = !STATE_MASK;

/// Number of positions to shift the ref count.
const REF_COUNT_SHIFT: usize = REF_COUNT_MASK.count_zeros() as usize;

/// One ref count.
const REF_ONE: usize = 1 << REF_COUNT_SHIFT;

/// State a task is initialized with.
///
/// A task is initialized with three references:
///
///  * A reference that will be stored in an `OwnedTasks` or `LocalOwnedTasks`.
///  * A reference that will be sent to the scheduler as an ordinary notification.
///  * A reference for the `JoinHandle`.
///
/// As the task starts with a `JoinHandle`, `JOIN_INTEREST` is set.
/// As the task starts with a `Notified`, `NOTIFIED` is set.
const INITIAL_STATE: usize = (REF_ONE * 3) | JOIN_INTEREST | NOTIFIED;
```

各个状态的含义在注释中已经进行了描述，与状态相关的在 [0 ~ 5] 位。与任务引用计数相关的位在 [6 ~ usize::BITS-1]位。

每个任务初始化时，有三个引用。状态被设置为 `(REF_ONE * 3) | JOIN_INTEREST | NOTIFIED`

- OwnedTasks 或 LocalOwnedTasks
- 在 scheduler 或者 notification 中
- 在 JoinHandle 中

在初始化时，如果任务有 JoinHandle，其状态中的 JOIN_INTEREST 位为 1；若在初始化时，如果任务有 Notified，其状态中的 NOTIFIED 位为 1；

所有对状态有关的操作通过于状态相关的 Snapshot 进行，针对 AtomicUsize 的操作，是通过参数为 Snapshot 的闭包（`FnMut(Snapshot) -> Option<Snapshot>`）进行的。闭包的参数为当前的状态，闭包根据当前的状态计算出要更新状态的结果，再通过 compare_exchange 设置任务新的状态，如果状态更新成功，则会返回闭包的结果；如果状态更新失败，则会返回当前的状态。

```rust
fn fetch_update_action<F, T>(&self, mut f: F) -> T
where
    F: FnMut(Snapshot) -> (T, Option<Snapshot>),
{
    let mut curr = self.load();

    loop {
        let (output, next) = f(curr);
        let next = match next {
            Some(next) => next,
            None => return output,
        };

        let res = self.val.compare_exchange(curr.0, next.0, AcqRel, Acquire);

        match res {
            Ok(_) => return output,
            Err(actual) => curr = Snapshot(actual),
        }
    }
}

fn fetch_update<F>(&self, mut f: F) -> Result<Snapshot, Snapshot>
where
    F: FnMut(Snapshot) -> Option<Snapshot>,
{
    let mut curr = self.load();

    loop {
        let next = match f(curr) {
            Some(next) => next,
            None => return Err(curr),
        };

        let res = self.val.compare_exchange(curr.0, next.0, AcqRel, Acquire);

        match res {
            Ok(_) => return Ok(next),
            Err(actual) => curr = Snapshot(actual),
        }
    }
}
```

封装的状态操作函数：

- transition_to_running：将任务转化为 running 状态。如果任务当前处于 Idle 状态（不处于 running 或者 complete）时，会设置 running 位，取消 notified 位，如果任务被取消了，则设置 action 为 Cancelled，否则 action 为 success。如果任务不处于 Idle 状态，会将引用计数 -1，如果任务的引用计数为 0，则将 action 设置为 Dealloc，如果引用计数不为 0，则转化为 running 状态失败，将 action 设置为 failed。
- transition_to_idle：将任务从 running 转化为 idle。任务必须是处于 running 状态，如果任务已经被设置为 cancelled 了，则不会更新状态，直接将 action 设置为 Cancelled 并返回。如果任务没有被 notified，则引用计数 -1，如果引用计数为 0，则设置 action 为 OkDealloc，需要回收，否则 action 设置为 OK。如果任务已经被 Notified 了，则引用计数 +1，将 action 设置为 OKNotified。
- transition_to_complete：任务必须处于 running 且不处于 complete
- transition_to_terminal：减小 count 个引用计数
- transition_to_notified_by_val：如果任务处于 running，则设置为 notified，不需要唤醒，并且将引用计数 -1；如果任务处于 complete 或者已经 notified，则引用计数 -1，若引用计数为 0，则需要回收；在其他状态下，任务状态设置为 notified，引用计数 +1；
- transition_to_notified_by_ref：操作任务的 ref 来进行 notifed，更新任务的状态，只有在任务既不处于 running 或 complete 或 notified 时，才会增加引用计数
- transition_to_notified_and_cancel：当任务处于 cancelled 或者 complete 时，无事发生；当任务处于 running 时，设置 notified 和 cancelled；否则，设置 cancelled，如果任务不处于 notified，则设置 notified，引用计数 +1
- transition_to_shutdown：如果任务处于 idle，则设置 running；否则直接设置 cancelled。
- drop_join_handle_fast：在 spawn 任务时，直接将 JoinHandle 对应的位清空，并将引用计数 -1
- transition_to_join_handle_dropped：取消任务状态的 join_interested。如果任务不处于 complete，则取消 join_waker
- set_join_waker：任务必须设置了 join_interested。如果任务已经 complete，则无事发生，否则，直接设置 join_waker
- unset_waker：任务必须设置了 join_interested 和 join_waker。如果任务已经 complete，则无事发生，否则，直接取消 join_waker
- unset_waker_after_complete：在任务 complete 和 join_waker 后，取消 join_waker
