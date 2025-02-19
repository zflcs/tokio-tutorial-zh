# IO

所有与 文件系统的交互都通过 spawn_blocking 函数进行，其中封装了 std 的相关操作，因为 std 已经提供了一层封装，所以不需要像在内核中直接使用 async 提供异步 IO，那样繁琐（因为 dyn trait 中使用 async 会导致类型不安全）
