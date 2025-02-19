# spawn_blocking

这个函数用于执行一些会阻塞的操作。

创建一个 oneshot 通道，将阻塞操作存储在 tx 端，并创建出一个任务，将该任务放入到一个局部队列（使用 tokio_thread_local 宏修饰）中，并返回包装了 oneshot rx 的 joinhandle。

与文件系统相关的操作：例如 canonicalize、copy、create_dir_all、create_dir、dir_builder、file、hard_link、metadata、read_dir、read_link、read_to_string 等等都使用 asyncify（spawn_blocking） 函数进行包装。
