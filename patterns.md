# 模式集 — TCP/IP 网络编程

## 迭代服务器模式
**When to use**: 客户端数量少、每次请求处理时间短
**How**: accept → 处理 → close → 循环回 accept
**Trade-offs**: 同时只能服务一个客户端，其他客户端需等待

## 多进程并发服务器模式
**When to use**: 需要同时服务多个客户端，Linux 平台
**How**: 父进程 accept 后 fork 子进程处理，子进程结束后父进程 wait/waitpid 回收；注册 SIGCHLD 信号处理避免僵尸进程
**Trade-offs**: 每个连接消耗一个进程，进程数过多时上下文切换开销大

## I/O 多路复用服务器模式 (select)
**When to use**: 连接数适中（< 1024），单进程需同时管理多个连接
**How**: 用 FD_SET 注册关心的文件描述符，调用 select 等待事件，返回后用 FD_ISSET 检查哪个 fd 有数据
**Trade-offs**: 单进程避免上下文切换；但 fd 数量受 FD_SETSIZE 限制，每次需全量传递 fd_set

## epoll 服务器模式
**When to use**: Linux 平台、高并发场景（千级连接）
**How**: epoll_create 创建实例 → epoll_ctl 注册关心的 fd 和事件类型 → epoll_wait 等待事件 → 处理返回的事件数组
**Trade-offs**: 内核维护事件表，O(1) 返回就绪 fd；仅 Linux 支持

## 多线程并发服务器模式
**When to use**: 需要同时服务多个客户端，线程开销小于进程
**How**: 主线程 accept 后创建工作线程处理连接；用 mutex/semaphore 保护共享数据；线程结束后 join 或 detach
**Trade-offs**: 比多进程轻量；但需处理线程同步和临界区问题

## IOCP 服务器模式
**When to use**: Windows 平台、高并发场景
**How**: 创建完成端口 → 关联套接字 → 投递异步 I/O 请求 → 工作线程从完成端口获取完成通知 → 处理结果
**Trade-offs**: Windows 最高效的 I/O 模型；编程复杂度较高

## 管道双向通信模式
**When to use**: 父子进程间需要双向数据交换
**How**: 创建 2 个管道（pipe A: 父→子，pipe B: 子→父），各关闭不需要的端口端
**Trade-offs**: 需 2 个管道才能实现真正的双向通信；单管道双向通信会因时序导致死锁

## 回声客户端 I/O 分割模式
**When to use**: 客户端需要同时收发数据，避免互相阻塞
**How**: 父进程负责接收数据，子进程负责发送数据（或用 select/多线程分割 I/O）
**Trade-offs**: 分割后逻辑清晰；但需额外进程/线程资源

## 应用层协议设计模式
**When to use**: 客户端与服务器需交换结构化数据
**How**: 定义数据边界（分隔符或长度前缀）→ 约定数据类型和顺序 → 实现打包/解包函数
**Trade-offs**: 显式边界避免 TCP 流式传输的粘包问题；协议设计增加复杂度

## 标准库 I/O 缓冲模式
**When to use**: 需要更高效的文件/套接字 I/O
**How**: 用 fdopen 将文件描述符包装为 FILE*，使用 fputs/fgets 等标准 I/O 函数；fflush 刷新缓冲区
**Trade-offs**: 标准 I/O 带缓冲减少系统调用次数；但需注意缓冲区与套接字半关闭的冲突

## 文件描述符复制模式
**When to use**: 分离 I/O 流后仍需访问同一资源
**How**: 用 dup/dup2 复制文件描述符，两个描述符指向同一文件表项，但可独立关闭
**Trade-offs**: 解决了流分离后的半关闭问题；但需管理多个描述符的生命周期

## 重叠 I/O 模式
**When to use**: Windows 平台需要异步 I/O
**How**: 初始化 WSAOVERLAPPED 结构体 → 调用 WSARecv/WSASend 并传入 OVERLAPPED → 用 WaitForSingleObject 或完成例程确认完成
**Trade-offs**: I/O 不阻塞线程，提高并发能力；需额外管理 OVERLAPPED 结构体

## 紧急数据 (OOB) 模式
**When to use**: 需要在普通数据流中插入高优先级消息
**How**: 用 send(sock, & urg, 1, MSG_OOB) 发送 → 接收端注册 SIGURG 信号 → 用 fcntl 设置属主进程 → 信号处理函数中用 recv(MSG_OOB) 读取
**Trade-offs**: 紧急消息优先于普通数据；但实际传输仍走 TCP，非真正"带外"

## 条件触发 vs 边缘触发
**When to use**: epoll 编程时选择触发模式
**How**: 条件触发(LT)：只要有数据就一直通知；边缘触发(ET)：仅状态变化时通知一次，需配合非阻塞 I/O 循环读取至 EAGAIN
**Trade-offs**: ET 效率更高但编程复杂，LT 更安全简单；高性能服务器推荐 ET
