# async_fd

文件类型必须是支持 OS poll 接口（poll、epoll、kqueue 等）的。必须可以设置成非阻塞模式。

AsyncFd 允许在 tokio Reactor 中直接注册文件描述符，允许直接等待文件描述符可读或可写。
