---
name: tcp-ip-network-programming
description: "Knowledge base from \"TCP/IP 网络编程\" by 尹圣雨 (Yoon Sung Woo). Use when applying socket programming frameworks for TCP/UDP server-client design, I/O multiplexing (select/epoll/IOCP), multi-process/multi-thread concurrency, process intercommunication, Windows overlapped I/O, and HTTP server implementation on Linux and Windows."
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# TCP/IP 网络编程
**Author**: 尹圣雨 (Yoon Sung Woo) | **Pages**: ~421 | **Chapters**: 25 | **Generated**: 2026-08-18

## How to Use This Skill

- **Without arguments** — load core frameworks for reference
- **With a topic** — ask about `select`, `epoll`, `IOCP`, `shutdown`, or another indexed topic; I find and read the relevant chapter
- **With chapter** — ask for `ch05`; I load that specific chapter
- **Browse** — ask "what chapters do you have?" to see the full index

When you ask about a topic not covered in Core Frameworks below, I will read
the relevant chapter file before answering.

---

## Core Frameworks & Mental Models

### 套接字编程四步法
服务器端必经的四步流程: `socket()` 创建套接字 → `bind()` 分配地址 → `listen()` 转为监听状态 → `accept()` 接受连接并返回新套接字。客户端只需两步: `socket()` + `connect()`。Use this as the mental model whenever designing any TCP server-client system.

### TCP/IP 四层协议栈
应用层 → 传输层(TCP/UDP) → 网络层(IP) → 链路层(以太网)。每层只与相邻层交互，上层不关心下层实现细节。Think of it as "每层只管自己的事"。

### 三次握手与四次断开
TCP 连接通过 SYN/SYN+ACK/ACK 三步建立；断开通过 FIN/ACK/FIN/ACK 四步完成。Time-wait 发生在主动断开方，持续约 2-4 分钟，期间端口不可复用。Use SO_REUSEADDR to bypass Time-wait port blocking on server restart.

### 流式 vs 数据报决策
TCP (SOCK_STREAM) = 面向连接、可靠传输、无数据边界 — 适合文件传输、Web。UDP (SOCK_DGRAM) = 无连接、不可靠、有数据边界 — 适合实时视频、DNS、多播。Prefer TCP when data integrity matters; prefer UDP when speed matters more than completeness.

### 字节序转换规则
网络字节序 = 大端序 (Big-Endian)。发送前用 htons/htonl 转换端口和地址；接收后用 ntohs/ntohl 转回主机序。Always convert before sending, always convert after receiving — never assume host byte order.

### 迭代 → 并发演进路径
迭代服务器(Ch4) → 多进程 fork(Ch10) → I/O 多路复用 select(Ch12) → epoll(Ch17) → 多线程(Ch18) → IOCP(Ch23)。Each step scales better but adds complexity. Choose based on expected connection count and platform.

### I/O 多路复用: select → epoll
select: 每次调用需全量传递 fd_set，上限 1024，O(n) 扫描。epoll: 内核维护事件表，epoll_wait 只返回就绪 fd，O(1)，无上限。Prefer epoll on Linux for > 1000 connections; select is fine for learning or small-scale.

### 条件触发 vs 边缘触发
条件触发(LT, epoll 默认): 只要有数据就持续通知。边缘触发(ET): 仅状态变化时通知一次，必须配合非阻塞 I/O 循环读取至 EAGAIN。Use ET for maximum performance; use LT when correctness is more important than raw speed.

### 半关闭 (Half-Close)
`close()` = 同时关闭收发。`shutdown(SHUT_WR)` = 只关闭发送(发 FIN)，保留接收能力。Use shutdown when you need to signal "I'm done sending" but still want to receive the other party's final response — e.g., file upload followed by server confirmation.

### 多进程并发模型 (Linux)
父进程 accept 后 fork 子进程处理每个连接。子进程退出后发 SIGCHLD，父进程用 waitpid 回收避免僵尸进程。Register SIGCHLD handler with sigaction and loop waitpid inside it to reap all pending children.

### 管道双向通信
单管道无法安全实现双向通信(时序会导致死锁)。Use 2 pipes: one for parent→child, one for child→parent. Close the unused end in each process to avoid resource leaks.

### 线程同步三件套
mutex 保护临界区(同一时间只允许一个线程访问)。semaphore 控制资源计数(允许多个线程访问但限数量)。Use mutex for mutual exclusion, semaphore for resource counting, condition variable for event waiting.

### IOCP 编程模型 (Windows)
CreateIoCompletionPort 创建完成端口 → 关联套接字 → 投递 WSARecv/WSASend 异步请求 → 工作线程 GetQueuedCompletionStatus 等待完成通知 → 处理结果后再次投递。This is the most scalable Windows I/O model — worker threads process completions from a queue, avoiding per-connection thread creation.

### 文件描述符与流分离
用 fdopen() 将套接字文件描述符包装为 FILE* 指针后可用标准 I/O 函数。用 dup/dup2 复制描述符实现流分离后的半关闭。Separate the "read FILE*" and "write FILE*" so shutdown can close one direction while the other stays open.

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-socket-fundamentals.md) | 理解网络编程和套接字 | socket/bind/listen/accept/connect, 文件描述符 |
| [ch02](chapters/ch02-socket-types-protocols.md) | 套接字类型与协议 | SOCK_STREAM, SOCK_DGRAM, 协议族 |
| [ch03](chapters/ch03-address-family-data-sequence.md) | 地址族与数据序列 | sockaddr_in, htons/htonl, inet_addr, INADDR_ANY |
| [ch04](chapters/ch04-tcp-server-client-1.md) | 基于 TCP 的服务器端/客户端(1) | TCP 握手, 迭代服务器, echo server |
| [ch05](chapters/ch05-tcp-server-client-2.md) | 基于 TCP 的服务器端/客户端(2) | 应用层协议设计, 粘包问题 |
| [ch06](chapters/ch06-udp-server-client.md) | 基于 UDP 的服务器端/客户端 | sendto/recvfrom, 已连接 UDP |
| [ch07](chapters/ch07-graceful-disconnect.md) | 优雅地断开套接字连接 | shutdown, 半关闭, SHUT_WR |
| [ch08](chapters/ch08-domain-name-dns.md) | 域名及网络地址 | DNS, gethostbyname, hostent |
| [ch09](chapters/ch09-socket-options.md) | 套接字的可选项 | getsockopt/setsockopt, SO_REUSEADDR, TCP_NODELAY |
| [ch10](chapters/ch10-multiprocess-server.md) | 多进程服务器端的实现 | fork, 僵尸进程, SIGCHLD, wait/waitpid |
| [ch11](chapters/ch11-interprocess-communication.md) | 进程间通信 | pipe, 单向/双向通信 |
| [ch12](chapters/ch12-io-multiplexing-select.md) | I/O 多路复用 | select, fd_set, FD_SET/FD_ISSET |
| [ch13](chapters/ch13-various-io-functions.md) | 多种 I/O 函数 | send/recv, MSG_OOB, MSG_PEEK, readv/writev |
| [ch14](chapters/ch14-multicast-broadcast.md) | 多播与广播 | TTL, IP_MULTICAST_TTL, IP_ADD_MEMBERSHIP, SO_BROADCAST |
| [ch15](chapters/ch15-socket-standard-io.md) | 套接字与标准 I/O | fopen/fputs/fgets, fdopen, fileno |
| [ch16](chapters/ch16-io-stream-separation.md) I/O 流分离 | dup/dup2, 文件描述符复制, 半关闭 |
| [ch17](chapters/ch17-epoll.md) | 优于 select 的 epoll | epoll_create/ctl/wait, 条件触发 vs 边缘触发 |
| [ch18](chapters/ch18-multithread-server.md) | 多线程服务器端的实现 | pthread_create, mutex, semaphore, 线程同步 |
| [ch19](chapters/ch19-windows-threads.md) | Windows 平台下线程的使用 | CreateThread, 内核对象, WaitForSingleObject |
| [ch20](chapters/ch20-windows-thread-sync.md) | Windows 平台下的线程同步 | CRITICAL_SECTION, Mutex, Event, Semaphore |
| [ch21](chapters/ch21-async-notify-io.md) | 异步通知 I/O 模型 | WSAAsyncSelect, 窗口消息 |
| [ch22](chapters/ch22-overlapped-io.md) | 重叠 I/O 模型 | WSAOVERLAPPED, WSARecv/WSASend |
| [ch23](chapters/ch23-iocp.md) | IOCP | CreateIoCompletionPort, GetQueuedCompletionStatus |
| [ch24](chapters/ch24-http-server.md) | 制作 HTTP 服务器端 | HTTP 协议, GET/POST, 状态码, Web 服务器 |
| [ch25](chapters/ch25-advanced-topics.md) | 进阶内容 | 学习路径, 相关书籍 |

## Topic Index

- **accept()** → ch01, ch04
- **bind()** → ch01, ch03
- **broadcast (广播)** → ch14
- **close vs shutdown** → ch07
- **connect()** → ch01, ch06
- **CRITICAL_SECTION** → ch20
- **DNS** → ch08
- **dup/dup2** → ch16
- **echo server** → ch04, ch05
- **epoll** → ch17
- **边缘触发 (ET)** → ch17
- **条件触发 (LT)** → ch17
- **fdopen** → ch15
- **fork()** → ch10
- **gethostbyname** → ch08
- **getsockopt/setsockopt** → ch09
- **半关闭 (half-close)** → ch07, ch16
- **htons/htonl** → ch03
- **HTTP** → ch24
- **INADDR_ANY** → ch03
- **IOCP** → ch23
- **I/O 多路复用** → ch12, ch17
- **迭代服务器** → ch04
- **listen()** → ch01
- **MSG_OOB** → ch13
- **MSG_PEEK** → ch13
- **多播 (multicast)** → ch14
- **mutex** → ch18
- **Nagle 算法** → ch09
- **pipe()** → ch11
- **pthread** → ch18
- **readv/writev** → ch13
- **recvfrom/sendto** → ch06
- **select()** → ch12
- **semaphore** → ch18
- **SIGCHLD** → ch10
- **sigaction** → ch10
- **socket()** → ch01, ch02
- **sockaddr_in** → ch03
- **SOCK_STREAM** → ch02
- **SOCK_DGRAM** → ch02
- **SO_REUSEADDR** → ch09
- **TCP 三次握手** → ch04
- **TCP_NODELAY** → ch09
- **Time-wait** → ch09
- **TTL** → ch14
- **UDP** → ch06
- **WSAAsyncSelect** → ch21
- **WSAOVERLAPPED** → ch22
- **僵尸进程** → ch10

## Supporting Files

- [glossary.md](glossary.md) — 所有关键术语与定义
- [patterns.md](patterns.md) — 所有技术模式与设计方法
- [cheatsheet.md](cheatsheet.md) — 决策规则与速查表

---

## Scope & Limits

This skill covers the book content only. For hands-on implementation in your codebase,
combine with project-specific tools. For topics beyond this book, check related skills
or ask the agent directly.
