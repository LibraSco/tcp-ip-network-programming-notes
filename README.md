
# TCP网络编程笔记 · tcp-ip-network-programming

![Book](https://img.shields.io/badge/Book-TCP%2FIP_%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-orange)
![Chapters](https://img.shields.io/badge/Chapters-25-blue)
![Language](https://img.shields.io/badge/Coverage-Linux%20%26%20Windows-green)
![Format](https://img.shields.io/badge/Format-book--to--skill-lightgrey)

> 基于韩国作者**尹圣雨（Yoon Sung Woo）**《TCP/IP 网络编程》一书，用 **book-to-skill** 方法沉淀的结构化网络编程知识库。
>
> A structured knowledge base distilled from *"TCP/IP 网络编程"* by **Yoon Sung Woo**, built with the **book-to-skill** methodology. Covers socket programming, TCP/UDP, I/O multiplexing (select / epoll / IOCP), multi-process & multi-thread concurrency, and HTTP server implementation on both Linux and Windows.

---

## 关于本项目 / About

本书是网络编程领域的经典入门与进阶读物。本仓库不是简单的读书笔记，而是把整本书（约 **421 页、25 章**）拆解为一份可被 AI 助手直接调用的**技能知识库（Skill）**：以一个 `SKILL.md` 作为索引入口，把每一章凝练为独立的章节文件，并抽出术语表、模式库、决策速查表三类支撑文件。

> This repository is not a plain reading-notes dump. The whole book (~421 pages, 25 chapters) is decomposed into an **AI-ready skill knowledge base**: a single `SKILL.md` acts as the index entry, each chapter is condensed into its own file, and three supporting files (glossary, patterns, cheatsheet) capture the reusable knowledge.

---

## 书籍信息 / Book Info

| 项目 | 内容 |
|------|------|
| 书名 | TCP/IP 网络编程 |
| 作者 | 尹圣雨 (Yoon Sung Woo) |
| 页数 | ~421 |
| 章节数 | 25 |
| 覆盖平台 | Linux（socket / select / epoll / fork / pthread）、Windows（线程 / 同步 / 异步通知 I/O / 重叠 I/O / IOCP） |
| 知识库生成日期 | 2026-08-18 |

---

## 目录结构 / Repository Structure

整个仓库就是一份 book-to-skill 知识库，**没有额外代码目录**，全部内容都是带锚点的 Markdown 笔记文件：

```
.
├── SKILL.md                # 知识库入口：核心框架 + 章节索引 + 主题索引
├── chapters/               # 25 个章节学习笔记（每章一个文件）
│   ├── ch01-socket-fundamentals.md
│   ├── ch02-socket-types-protocols.md
│   ├── ch03-address-family-data-sequence.md
│   ├── ch04-tcp-server-client-1.md
│   ├── ch05-tcp-server-client-2.md
│   ├── ch06-udp-server-client.md
│   ├── ch07-graceful-disconnect.md
│   ├── ch08-domain-name-dns.md
│   ├── ch09-socket-options.md
│   ├── ch10-multiprocess-server.md
│   ├── ch11-interprocess-communication.md
│   ├── ch12-io-multiplexing-select.md
│   ├── ch13-various-io-functions.md
│   ├── ch14-multicast-broadcast.md
│   ├── ch15-socket-standard-io.md
│   ├── ch16-io-stream-separation.md
│   ├── ch17-epoll.md
│   ├── ch18-multithread-server.md
│   ├── ch19-windows-threads.md
│   ├── ch20-windows-thread-sync.md
│   ├── ch21-async-notify-io.md
│   ├── ch22-overlapped-io.md
│   ├── ch23-iocp.md
│   ├── ch24-http-server.md
│   └── ch25-advanced-topics.md
├── glossary.md             # 关键术语与定义
├── patterns.md             # 技术模式与设计方法
└── cheatsheet.md           # 决策规则与速查表
```

---

## 如何使用 / How to Use

这是一份可被 AI 编程助手加载调用的 Skill。使用方式有三种：

- **不带参数**：直接加载核心框架（`SKILL.md` 中的 Core Frameworks）用于参考。
- **带主题**：询问 `select`、`epoll`、`IOCP`、`shutdown` 等已索引主题，助手定位并读取对应章节后作答。
- **带章节号**：直接索要 `ch05`，加载指定章节。
- **浏览**：问 "what chapters do you have?" 查看完整索引。

> 当你提问的主题未出现在核心框架中时，助手会先读取相关章节文件再回答。

---

## 核心框架速览 / Core Frameworks

`SKILL.md` 顶部沉淀了若干可直接复用的心智模型，举其要者：

- **套接字编程四步法**：`socket() → bind() → listen() → accept()`；客户端只需 `socket() + connect()`。
- **三次握手与四次断开**：SYN/SYN+ACK/ACK 建立；FIN/ACK/FIN/ACK 断开；Time-wait 出现在主动断开方，用 `SO_REUSEADDR` 规避端口占用。
- **流式 vs 数据报**：TCP（`SOCK_STREAM`）可靠无边界，UDP（`SOCK_DGRAM`）不可靠有边界，按"完整性优先"还是"速度优先"决策。
- **字节序转换**：网络序=大端；发送前 `htons/htonl`，接收后 `ntohs/ntohl`。
- **迭代→并发演进**：迭代服务器 → fork → select → epoll → 多线程 → IOCP。
- **条件触发 vs 边缘触发**：LT 持续通知，ET 仅状态变化通知一次，需配合非阻塞 I/O。
- **半关闭**：`shutdown(SHUT_WR)` 只关发送、保留接收，区别于 `close()` 同时关收发。

---

## 章节索引 / Chapter Index

| # | 标题 | 关键框架 |
|---|------|----------|
| [ch01](chapters/ch01-socket-fundamentals.md) | 理解网络编程和套接字 | socket/bind/listen/accept/connect、文件描述符 |
| [ch02](chapters/ch02-socket-types-protocols.md) | 套接字类型与协议 | SOCK_STREAM、SOCK_DGRAM、协议族 |
| [ch03](chapters/ch03-address-family-data-sequence.md) | 地址族与数据序列 | sockaddr_in、htons/htonl、inet_addr、INADDR_ANY |
| [ch04](chapters/ch04-tcp-server-client-1.md) | 基于 TCP 的服务器端/客户端(1) | TCP 握手、迭代服务器、echo server |
| [ch05](chapters/ch05-tcp-server-client-2.md) | 基于 TCP 的服务器端/客户端(2) | 应用层协议设计、粘包问题 |
| [ch06](chapters/ch06-udp-server-client.md) | 基于 UDP 的服务器端/客户端 | sendto/recvfrom、已连接 UDP |
| [ch07](chapters/ch07-graceful-disconnect.md) | 优雅地断开套接字连接 | shutdown、半关闭、SHUT_WR |
| [ch08](chapters/ch08-domain-name-dns.md) | 域名及网络地址 | DNS、gethostbyname、hostent |
| [ch09](chapters/ch09-socket-options.md) | 套接字的可选项 | getsockopt/setsockopt、SO_REUSEADDR、TCP_NODELAY |
| [ch10](chapters/ch10-multiprocess-server.md) | 多进程服务器端的实现 | fork、僵尸进程、SIGCHLD、waitpid |
| [ch11](chapters/ch11-interprocess-communication.md) | 进程间通信 | pipe、单向/双向通信 |
| [ch12](chapters/ch12-io-multiplexing-select.md) | I/O 多路复用 | select、fd_set、FD_SET/FD_ISSET |
| [ch13](chapters/ch13-various-io-functions.md) | 多种 I/O 函数 | send/recv、MSG_OOB、MSG_PEEK、readv/writev |
| [ch14](chapters/ch14-multicast-broadcast.md) | 多播与广播 | TTL、IP_MULTICAST_TTL、IP_ADD_MEMBERSHIP、SO_BROADCAST |
| [ch15](chapters/ch15-socket-standard-io.md) | 套接字与标准 I/O | fdopen、fileno、fputs/fgets |
| [ch16](chapters/ch16-io-stream-separation.md) | I/O 流分离 | dup/dup2、文件描述符复制、半关闭 |
| [ch17](chapters/ch17-epoll.md) | 优于 select 的 epoll | epoll_create/ctl/wait、条件触发 vs 边缘触发 |
| [ch18](chapters/ch18-multithread-server.md) | 多线程服务器端的实现 | pthread_create、mutex、semaphore、线程同步 |
| [ch19](chapters/ch19-windows-threads.md) | Windows 平台下线程的使用 | CreateThread、内核对象、WaitForSingleObject |
| [ch20](chapters/ch20-windows-thread-sync.md) | Windows 平台下的线程同步 | CRITICAL_SECTION、Mutex、Event、Semaphore |
| [ch21](chapters/ch21-async-notify-io.md) | 异步通知 I/O 模型 | WSAAsyncSelect、窗口消息 |
| [ch22](chapters/ch22-overlapped-io.md) | 重叠 I/O 模型 | WSAOVERLAPPED、WSARecv/WSASend |
| [ch23](chapters/ch23-iocp.md) | IOCP | CreateIoCompletionPort、GetQueuedCompletionStatus |
| [ch24](chapters/ch24-http-server.md) | 制作 HTTP 服务器端 | HTTP 协议、GET/POST、状态码、Web 服务器 |
| [ch25](chapters/ch25-advanced-topics.md) | 进阶内容 | 学习路径、相关书籍 |

---

## 支撑文件 / Supporting Files

| 文件 | 用途 |
|------|------|
| [glossary.md](glossary.md) | 全书关键术语与定义汇总 |
| [patterns.md](patterns.md) | 可复用的技术模式与设计方法 |
| [cheatsheet.md](cheatsheet.md) | 决策规则与 API 速查表 |

---

## book-to-skill 方法论 / Methodology

本知识库遵循 book-to-skill 的四条核心原则：

1. **原子性 Atomic**：一章一文件，每个 Markdown 自带锚点，可被独立检索与调用。
2. **可验证 Verifiable**：核心框架给出明确决策规则（如 LT/ET 何时用、TCP/UDP 如何选）。
3. **可检索 Searchable**：`SKILL.md` 同时维护章节索引与主题索引，按主题即可定位到对应章节。
4. **渐进式 Progressive**：从单章框架到术语表、模式库、速查表，知识逐层沉淀，按需展开。

---

## 相关资源 / Resources

- 原书：尹圣雨《TCP/IP 网络编程》
- RFC 文档：[rfc-editor.org](https://www.rfc-editor.org/)
- Linux man pages：socket(2)、bind(2)、listen(2)、accept(2)、select(2)、epoll(7)
- Windows 套接字文档：Winsock / IOCP（Microsoft Learn）

---

## 说明 / Note

本仓库为个人学习整理，笔记结构与内容凝练自原书，仅供学习交流使用，请支持正版图书。
