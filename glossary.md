# 术语表 — TCP/IP 网络编程

**ACK (Acknowledgment)** — TCP 确认应答，接收方告知发送方数据已到达 (Ch 4)

**bind()** — 分配地址信息（IP + 端口）给套接字的函数 (Ch 1, Ch 3)

**broadcast (广播)** — 一次性向同一网络内所有主机发送数据的方式，基于 UDP (Ch 14)

**close()** — 完全关闭套接字连接，同时切断输入和输出流 (Ch 7)

**Completion Port (完成端口)** — Windows IOCP 模型的核心机制，用于高效处理大量并发 I/O (Ch 23)

**connect()** — 客户端向服务器端发起连接请求的函数 (Ch 1)

**CRITICAL_SECTION** — Windows 用户模式线程同步对象，轻量级互斥 (Ch 20)

**DNS (Domain Name System)** — 域名系统，将人类可读的域名转换为 IP 地址 (Ch 8)

**dup / dup2()** — 复制文件描述符的函数，用于 I/O 流分离 (Ch 16)

**echo server (回声服务器)** — 将客户端发送的数据原样返回的服务器，教学经典示例 (Ch 4, Ch 5)

**epoll** — Linux 高效 I/O 多路复用机制，优于 select (Ch 17)

**epoll_create / epoll_ctl / epoll_wait** — epoll 的创建、注册、等待三个核心函数 (Ch 17)

**Edge Trigger (边缘触发, ET)** — epoll 模式之一，仅在状态变化时通知，需配合非阻塞 I/O (Ch 17)

**fd_set** — select 函数使用的文件描述符集合类型 (Ch 12)

**FD_ZERO / FD_SET / FD_CLR / FD_ISSET** — 操作 fd_set 的四个位操作宏 (Ch 12)

**FIN** — TCP 连接断开请求包，四步断开的第一步 (Ch 4, Ch 7)

**fork()** — 创建子进程的系统调用，用于多进程服务器 (Ch 10)

**gethostbyname()** — 通过域名获取 IP 地址的函数 (Ch 8)

**getsockopt / setsockopt** — 读取/设置套接字可选项的函数 (Ch 9)

**half-close (半关闭)** — 只关闭输出流（发送 EOF）但保留接收能力的状态 (Ch 7)

**htons / htonl** — 主机字节序转网络字节序（16/32 位）(Ch 3)

**INADDR_ANY** — 特殊地址值，让服务器端自动获取本机 IP (Ch 3)

**IOCP (I/O Completion Port)** — Windows 最高效的异步 I/O 模型 (Ch 23)

**I/O 多路复用 (I/O Multiplexing)** — 通过一个进程同时监控多个文件描述符的技术 (Ch 12)

**ip_mreq** — 多播组加入/退出使用的结构体 (Ch 14)

**IP_ADD_MEMBERSHIP** — 加入多播组的套接字选项 (Ch 14)

**IP_MULTICAST_TTL** — 设置多播 TTL 值的套接字选项 (Ch 14)

**iterative server (迭代服务器)** — 同一时间只能服务一个客户端的服务器 (Ch 4)

**Level Trigger (条件触发, LT)** — epoll 默认模式，只要有数据就持续通知 (Ch 17)

**listen()** — 将套接字转为可接收连接状态的函数 (Ch 1)

**MSG_OOB** — 紧急消息发送选项 (Ch 13)

**MSG_PEEK** — 窥探接收缓冲区数据但不删除的选项 (Ch 13)

**Mutex (互斥锁)** — 线程同步机制，保护临界区 (Ch 18)

**Nagle 算法** — TCP 默认启用的数据包合并算法，可通过 TCP_NODELAY 禁用 (Ch 9)

**ntohs / ntohl** — 网络字节序转主机字节序（16/32 位）(Ch 3)

**Overlapped I/O (重叠 I/O)** — Windows 异步 I/O 模型，I/O 操作与主线程重叠执行 (Ch 22)

**pipe()** — 创建管道用于进程间通信的函数 (Ch 11)

**pthread_create / pthread_join / pthread_detach** — 线程创建/回收/分离函数 (Ch 18)

**readv / writev** — 集中输入/分散输出函数，操作多个缓冲区 (Ch 13)

**recvfrom / sendto** — UDP 数据收发函数 (Ch 6)

**select()** — I/O 多路复用函数，监控多个文件描述符状态变化 (Ch 12)

**sem_wait / sem_post** — 信号量等待/释放操作 (Ch 18)

**SIGCHLD** — 子进程终止时发送给父进程的信号 (Ch 10)

**sigaction** — 信号注册函数，优于 signal() (Ch 10)

**shutdown()** — 半关闭函数，可选择关闭输入或输出流 (Ch 7)

**SO_REUSEADDR** — 地址再分配选项，解决 Time-wait 导致的端口占用 (Ch 9)

**SOCK_DGRAM** — 面向数据报的套接字类型，对应 UDP (Ch 2)

**SOCK_STREAM** — 面向连接的流式套接字类型，对应 TCP (Ch 2)

**SO_BROADCAST** — 启用广播功能的套接字选项 (Ch 14)

**socket()** — 创建套接字的函数，参数为协议族 + 类型 + 协议 (Ch 1)

**sockaddr_in** — IPv4 地址信息结构体 (Ch 3)

**TCP_NODELAY** — 禁用 Nagle 算法的套接字选项 (Ch 9)

**Time-wait** — TCP 断开后端口保持占用的时间段 (Ch 9)

**TTL (Time To Live)** — 数据包存活跳数，多播中控制传播范围 (Ch 14)

**UDP (User Datagram Protocol)** — 无连接、不可靠但高效的数据传输协议 (Ch 6)

**WSAAsyncSelect** — Windows 异步通知 I/O 模型，基于窗口消息 (Ch 21)

**WSAOVERLAPPED** — 重叠 I/O 结构体，记录异步 I/O 操作状态 (Ch 22)

**WSARecv / WSASend** — Windows 重叠 I/O 收发函数 (Ch 22)

**zombie process (僵尸进程)** — 已终止但未被父进程回收资源的进程 (Ch 10)
