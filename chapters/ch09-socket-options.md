# Chapter 9: 套接字的可选项

## Core Idea
套接字具有多种可配置的选项,通过getsockopt读取和setsockopt设置;其中SO_REUSEADDR解决Time-wait状态导致的地址重分配错误,TCP_NODELAY控制Nagle算法的开关以平衡网络效率与传输速度。

## Frameworks Introduced
- **套接字选项读写框架(getsockopt/setsockopt)**: 套接字选项按协议层(SOL_SOCKET/IPPROTO_TCP等)和选项名(SO_REUSEADDR/SO_RCVBUF等)组织,通过getsockopt读取当前值,通过setsockopt设置新值。
  - When to use: 需要查看或修改套接字的I/O缓冲大小、地址重用、Nagle算法等行为时
  - How: getsockopt(sock, level, optname, &optval, &optlen)读取;setsockopt(sock, level, optname, &optval, optlen)设置

- **SO_REUSEADDR地址重分配框架**: 服务器端先断开连接时套接字进入Time-wait状态(约数分钟),期间无法用同一端口重新bind;设置SO_REUSEADDR为TRUE可跳过Time-wait限制,立即重用端口。
  - When to use: 服务器端需要紧急重启(如故障后)且端口处于Time-wait状态时
  - How: bind之前调用setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &option(TRUE), sizeof(option))

- **Nagle算法控制框架(TCP_NODELAY)**: Nagle算法默认开启,通过缓冲数据等待ACK来减少网络数据包数量;传输大文件等场景下禁用Nagle可提升速度,通过TCP_NODELAY选项控制。
  - When to use: 传输大文件(不需等待ACK,缓冲很快填满)或需要低延迟传输时禁用Nagle;一般网络传输保持开启
  - How: setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &optval(1=禁用), sizeof(optval))

## Key Concepts
- **getsockopt**: `int getsockopt(int sock, int level, int optname, void *optval, socklen_t *optlen)`,读取套接字选项的当前值
- **setsockopt**: `int setsockopt(int sock, int level, int optname, const void *optval, socklen_t optlen)`,设置套接字选项的新值
- **SOL_SOCKET**: 通用套接字层,包含SO_TYPE/SO_SNDBUF/SO_RCVBUF/SO_REUSEADDR等选项
- **IPPROTO_TCP**: TCP协议层,包含TCP_NODELAY选项
- **SO_TYPE**: 查看套接字类型(只读),返回SOCK_STREAM或SOCK_DGRAM
- **SO_SNDBUF/SO_RCVBUF**: 输出/输入缓冲大小,可读写
- **SO_REUSEADDR**: 地址再分配选项,设为TRUE时允许重用Time-wait状态的端口
- **Time-wait状态**: 先断开连接(先发FIN)的套接字在四次握手后进入的等待状态,持续数分钟
- **TCP_NODELAY**: 禁用Nagle算法的选项,设为TRUE时关闭Nagle
- **Nagle算法**: 为防止数据包过多导致网络过载,只有收到前一数据的ACK后才发送下一数据的算法(1984年)

## Mental Models
- Use "options are organized by protocol layer then option name" when navigating socket options: level picks the layer (SOL_SOCKET/IPPROTO_TCP), optname picks the specific option
- Use "Time-wait protects the last ACK" when understanding why it exists: if the final ACK is lost, the Time-wait socket can retransmit it; but it blocks port reuse
- Use "SO_REUSEADDR overrides Time-wait for server restarts" when servers need to restart on the same port immediately
- Use "Nagle ON = fewer packets but slower; Nagle OFF = more packets but faster" when deciding TCP_NODELAY: default ON for network efficiency, OFF for large file transfer or low latency

## Anti-patterns
- **服务器未设置SO_REUSEADDR导致重启失败**: 服务器端先断开连接时进入Time-wait,重新运行同一端口会报"bind() error"——应在bind前设置SO_REUSEADDR
- **盲目禁用Nagle算法**: 禁用Nagle会增加数据包数量(即使1字节数据也有几十字节头),增加网络负载——仅在传输大文件等特定场景禁用
- **假设setsockopt设置的缓冲大小100%生效**: 系统会谨慎处理缓冲大小设置(不能设为0,需保留流控制和重传空间),实际值可能与请求不完全一致

## Code Examples
```c
// reuseadr_eserver.c —— 设置SO_REUSEADDR解决地址重分配
int serv_sock, clnt_sock;
int option, str_len;
socklen_t optlen;
struct sockaddr_in serv_adr, clnt_adr;

serv_sock = socket(PF_INET, SOCK_STREAM, 0);

optlen = sizeof(option);
option = TRUE;
setsockopt(serv_sock, SOL_SOCKET, SO_REUSEADDR, (void*)&option, optlen);  // 设置地址重用

memset(&serv_adr, 0, sizeof(serv_adr));
serv_adr.sin_family = AF_INET;
serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
serv_adr.sin_port = htons(atoi(argv[1]));

bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));  // 即使Time-wait也能bind
listen(serv_sock, 5);
// ... accept, read/write ...
```
- **What it demonstrates**: 在bind之前设置SO_REUSEADDR为TRUE,使服务器在Time-wait状态下也能立即重新绑定端口

```c
// get_buf.c / set_buf.c —— 读取和设置I/O缓冲大小
int sock;
int snd_buf, rcv_buf;
int state;
socklen_t len;

sock = socket(PF_INET, SOCK_STREAM, 0);

// 读取当前缓冲大小
len = sizeof(snd_buf);
state = getsockopt(sock, SOL_SOCKET, SO_SNDBUF, (void*)&snd_buf, &len);
len = sizeof(rcv_buf);
state = getsockopt(sock, SOL_SOCKET, SO_RCVBUF, (void*)&rcv_buf, &len);
// 默认: Input buffer=87380, Output buffer=16384

// 设置缓冲大小为3KB
snd_buf = 1024*3; rcv_buf = 1024*3;
setsockopt(sock, SOL_SOCKET, SO_RCVBUF, (void*)&rcv_buf, sizeof(rcv_buf));
setsockopt(sock, SOL_SOCKET, SO_SNDBUF, (void*)&snd_buf, sizeof(snd_buf));
// 设置后读取: Input=6144, Output=6144(大致反映但不完全一致)
```
- **What it demonstrates**: getsockopt读取和setsockopt设置I/O缓冲大小,系统谨慎处理实际值

## Reference Tables

| 协议层(level) | 选项名(optname) | 说明 | 读写 |
|---------------|----------------|------|------|
| SOL_SOCKET | SO_TYPE | 套接字类型 | 只读 |
| SOL_SOCKET | SO_SNDBUF | 输出缓冲大小 | 读写 |
| SOL_SOCKET | SO_RCVBUF | 输入缓冲大小 | 读写 |
| SOL_SOCKET | SO_REUSEADDR | 地址再分配 | 读写 |
| SOL_SOCKET | SO_KEEPALIVE | 保持连接 | 读写 |
| SOL_SOCKET | SO_BROADCAST | 广播 | 读写 |
| SOL_SOCKET | SO_DONTROUTE | 不路由 | 读写 |
| SOL_SOCKET | SO_OOBINLINE | 带外数据 | 读写 |
| SOL_SOCKET | SO_ERROR | 错误 | 只读 |
| IPPROTO_TCP | TCP_NODELAY | 禁用Nagle算法 | 读写 |

| 参数 | getsockopt | setsockopt |
|------|-----------|-----------|
| sock | 要查看的套接字fd | 要更改的套接字fd |
| level | 协议层 | 协议层 |
| optname | 选项名 | 选项名 |
| optval | 保存查看结果的缓冲(地址) | 保存要更改信息的缓冲(地址) |
| optlen | 第4参数缓冲大小(指针,返回实际大小) | 第4参数的信息字节数(值) |

| 场景 | Nagle算法 | 理由 |
|------|----------|------|
| 一般网络传输 | ON(默认) | 减少数据包数量,提高网络效率 |
| 传输大文件 | OFF | 缓冲快速填满,无需等待ACK,连续传输更快 |
| 低延迟传输 | OFF | 避免等待ACK的延迟 |

## Key Takeaways
1. 套接字选项通过getsockopt(读取)和setsockopt(设置)操作,按协议层(level)和选项名(optname)定位
2. SO_REUSEADDR设为TRUE允许服务器在Time-wait状态下重新绑定端口——服务器端应在bind前设置此项以支持紧急重启
3. Time-wait状态只有先断开连接(先发FIN)的套接字才会经历,持续数分钟,目的是防止最后的ACK丢失时无法重传
4. 客户端套接字端口号是动态分配的,无需关注Time-wait问题;服务器端固定端口需要关注
5. Nagle算法默认开启,通过TCP_NODELAY禁用——传输大文件时禁用可提升速度,一般场景保持开启以减少网络负载

## Connects To
- **Ch 4**: TCP三次握手和四次握手在第4章,本章的Time-wait基于四次握手过程
- **Ch 7**: close断开连接在第7章,本章讨论断开后的Time-wait状态和端口重用
- **Ch 5**: TCP的I/O缓冲概念在第5章补充,本章通过SO_SNDBUF/SO_RCVBUF选项读写缓冲大小
- **RFC 1122**: Time-wait状态的持续时间和TCP行为由RFC标准定义
