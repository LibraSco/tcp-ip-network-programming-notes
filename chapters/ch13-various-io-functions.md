# Chapter 13: 多种 I/O 函数(send & recv 的可选项、readv & writev)

## Core Idea
`send`/`recv` 比 `write`/`read` 多一个 `flags` 参数,可通过位或运算同时传递多个可选项(如 MSG_OOB 紧急消息、MSG_PEEK 不删除读、MSG_DONTROUTE 不路由);而 `readv`/`writev` 能对分散在多个缓冲的数据进行整合收发,减少 I/O 函数调用次数。

## Frameworks Introduced
- **send & recv 函数**: 带可选项的数据收发函数,`send(sockfd, buf, nbytes, flags)` 发送、`recv(sockfd, buf, nbytes, flags)` 接收;flags 用位或(bit OR)同时指定多个选项。
  - When to use: 需要紧急传输、窥探缓冲、不路由、非阻塞等特殊收发行为时。
  - How: 选用表 13-1 中的可选项,用 `|` 组合传入 flags;如 `MSG_PEEK|MSG_DONTWAIT`。
- **MSG_OOB 紧急消息机制**: 发送端 `send(..., MSG_OOB)` 标记紧急数据;接收端操作系统产生 `SIGURG` 信号,调用信号处理函数用 `recv(..., MSG_OOB)` 读取最后一个(1 字节)紧急数据。
  - When to use: 需要为重要/紧急数据建立单独处理方法和通道(如急诊患者优先)时。
  - How: Sender 调用 send 时指定 MSG_OOB → Receiver 用 fcntl(F_SETOWN) 设置套接字拥有者进程 → sigaction 注册 SIGURG 处理函数 → 在处理函数内 recv(..., MSG_OOB) 读取紧急数据。
- **MSG_PEEK 窥探读**: 读取输入缓冲数据但不删除,常与 MSG_DONTWAIT 组合,以非阻塞方式验证缓冲中是否存在待接收数据。
  - When to use: 需要在不消耗数据的前提下检查接收缓冲是否有数据(如等待完整数据到齐)。
  - How: `recv(sock, buf, size, MSG_PEEK|MSG_DONTWAIT)`,数据保留在缓冲,稍后可再次读取。
- **readv & writev(分散/聚集 I/O)**: 对分散在多个缓冲的数据进行整合传输(发送)与分别接收(读入)的函数,可一次调用替代多次 write/read,提高效率。
  - When to use: 待传输/接收数据分别位于不同缓冲(数组),需要减少 I/O 调用次数时;适用于所有该场景。
  - How: 构造 `struct iovec` 数组(每元素含 iov_base 缓冲地址 + iov_len 大小)→ 调用 `writev(fd, vec, n)`/`readv(fd, vec, n)`。

## Key Concepts
- **send 函数**: `ssize_t send(int sockfd, const void* buf, size_t nbytes, int flags)`,成功返回发送字节数,失败返回 -1。
- **recv 函数**: `ssize_t recv(int sockfd, void* buf, size_t nbytes, int flags)`,成功返回接收字节数(收到 EOF 返回 0),失败返回 -1。
- **flags 可选项**: send/recv 第四参数,用位或运算 `|` 同时传递多个信息。
- **MSG_OOB**: 用于传输带外数据(Out-of-band data),即紧急消息。
- **SIGURG 信号**: 收到 MSG_OOB 紧急消息时操作系统产生的信号,触发已注册的信号处理函数。
- **fcntl(F_SETOWN)**: 将套接字"拥有者"改为指定进程,使该套接字引发的 SIGURG 信号由该进程处理;`fcntl(recv_sock, F_SETOWN, getpid())`。
- **MSG_PEEK**: 验证输入缓冲中是否存在接收数据,读取后不删除数据。
- **MSG_DONTROUTE**: 数据传输时不参照路由表,在本地网络中寻找目的地。
- **MSG_DONTWAIT**: 非阻塞(Non-blocking)I/O。
- **MSG_WAITALL**: 防止函数返回,直到接收全部请求字节数。
- **writev 函数**: `ssize_t writev(int filedes, const struct iovec* iov, int iovcnt)`,把分散在多个缓冲的数据整合发送。
- **readv 函数**: `ssize_t readv(int filedes, const struct iovec* iov, int iovcnt)`,把接收数据分别读入多个缓冲。
- **struct iovec**: `{void* iov_base; size_t iov_len;}`,分别保存待收发数据的缓冲地址与大小。
- **TCP 不存在真正的带外数据**: MSG_OOB 不会加快传输速度,且每次只传递 1 字节紧急数据,剩余数据仍需普通 recv 读取。

## Mental Models
- Use **MSG_OOB** when 需要让紧急数据走特殊通道、被优先处理(信号驱动),而非排在普通数据流之后。
- Use **fcntl(F_SETOWN, getpid())** when 接收 SIGURG 信号前必须指定哪个进程负责处理该套接字的紧急消息。
- Use **MSG_PEEK + MSG_DONTWAIT** when 需非阻塞地检查缓冲是否有数据、又不消耗它(以便之后正式读取)。
- Use **writev/readv** when 数据分散在多个缓冲,可用一次调用替代多次 write/read,减少 I/O 调用次数、提高效率。

## Anti-patterns
- **误以为 MSG_OOB 能加快传输速度**: TCP 不存在真正的带外数据,MSG_OOB 不会提高传输速率,且每次仅 1 字节为紧急数据,剩余数据要靠普通 recv 读取。
- **不调用 fcntl(F_SETOWN) 就期望收到 SIGURG**: 若未把套接字拥有者设为本进程,SIGURG 不会送到本进程,紧急消息无法被处理。
- **所有数据都强行用 readv/writev**: 该函数在所有"数据分散于多缓冲"场景都适用,但若数据本就在单缓冲,反而增加 iovec 构造的复杂度。

## Code Examples
```c
/* oob_send.c —— 用 MSG_OOB 发送紧急消息 */
write(sock, "123", strlen("123"));
send(sock, "4", strlen("4"), MSG_OOB);      /* 紧急传输 '4' */
write(sock, "567", strlen("567"));
send(sock, "890", strlen("890"), MSG_OOB);  /* 紧急传输 '890' 的最后字节 */
close(sock);
```
```c
/* oob_recv.c —— 接收紧急消息(信号驱动) */
act.sa_handler=urg_handler;
sigemptyset(&act.sa_mask);
act.sa_flags=0;
/* ... bind / listen / accept 得到 recv_sock ... */
fcntl(recv_sock, F_SETOWN, getpid());        /* 关键:指定 SIGURG 处理进程 */
sigaction(SIGURG, &act, 0);

while((str_len=recv(recv_sock, buf, sizeof(buf), 0))!= 0) { /* 普通数据 */ }

void urg_handler(int signo)                 /* 紧急数据到达时被调用 */
{
    int str_len;
    char buf[BUF_SIZE];
    str_len=recv(recv_sock, buf, sizeof(buf)-1, MSG_OOB);  /* 读紧急数据 */
    buf[str_len]=0;
    printf("Urgent message: %s \n", buf);
}
```
```c
/* writev.c —— 整合分散缓冲的数据一次性发送 */
struct iovec vec[2];
char buf1[]="ABCDEFG";
char buf2[]="1234567";
int str_len;

vec[0].iov_base=buf1;
vec[0].iov_len=3;            /* 只发 "ABC" */
vec[1].iov_base=buf2;
vec[1].iov_len=4;            /* 只发 "1234" */

str_len=writev(1, vec, 2);   /* fd=1 标准输出;输出 "ABC1234" */
printf("Write bytes: %d \n", str_len);   /* 7 */
```
- **What it demonstrates**: MSG_OOB 用 send 发、SIGURG+fcntl+recv 信号驱动收;writev 用 iovec 数组把多缓冲数据整合成一次输出。

## Reference Tables

表 13-1 send & recv 的可选项及含义

| 可选项(Option) | 含义 | send | recv |
|----------------|------|------|------|
| MSG_OOB | 用于传输带外数据(紧急消息) | 可 | 可 |
| MSG_PEEK | 验证输入缓冲是否存在接收数据 | — | 可 |
| MSG_DONTROUTE | 不参照路由表,本地网络寻目的 | 可 | — |
| MSG_DONTWAIT | 非阻塞 I/O | 可 | 可 |
| MSG_WAITALL | 直到接收全部请求字节数才返回 | — | 可 |

注:不同操作系统对可选项支持不同,使用前需了解目标系统。

| 函数 | 签名 | 作用 |
|------|------|------|
| `send` | `ssize_t send(int sockfd, const void* buf, size_t nbytes, int flags)` | 带 flags 发送 |
| `recv` | `ssize_t recv(int sockfd, void* buf, size_t nbytes, int flags)` | 带 flags 接收 |
| `writev` | `ssize_t writev(int filedes, const struct iovec* iov, int iovcnt)` | 整合多缓冲发送 |
| `readv` | `ssize_t readv(int filedes, const struct iovec* iov, int iovcnt)` | 分别读入多缓冲 |
| `fcntl(F_SETOWN)` | `fcntl(sock, F_SETOWN, pid)` | 设置套接字拥有者进程 |

| struct iovec 成员 | 含义 |
|------------------|------|
| `iov_base` | 缓冲地址 |
| `iov_len` | 缓冲大小(发送字节数/接收最大字节数) |

## Key Takeaways
1. send/recv 比 write/read 多一个 flags 参数,用位或运算 `|` 同时指定多个可选项,实现紧急传输、窥探、不路由、非阻塞等行为。
2. MSG_OOB 不加快传输:TCP 无真正带外数据,每次仅 1 字节为紧急数据经 SIGURG 信号处理函数读取,剩余数据仍由普通 recv 读取。
3. 接收 MSG_OOB 紧急消息三件套:fcntl(F_SETOWN) 指定拥有者进程 → sigaction 注册 SIGURG → 在处理函数内 recv(..., MSG_OOB)。
4. MSG_PEEK 读取不删除缓冲数据,常配 MSG_DONTWAIT 非阻塞验证缓冲是否有数据,可对同一份数据多次读取。
5. writev/readv 通过 iovec 结构体数组,把分散在多缓冲的数据整合发送或分别接收,一次调用替代多次 write/read,在数据分散场景普遍适用、能提高效率。
6. 各可选项在不同操作系统中支持不同,实际开发需了解所用系统;本书重点讲不受系统差异影响的部分。

## Connects To
- **Ch 10**: 接收 MSG_OOB 的 SIGURG 信号处理沿用了第 10 章的 sigaction 信号机制。
- **Ch 12**: MSG_OOB 紧急数据到达也可通过 select 的 exceptset 监视"异常"事件捕获,二者机制相关。
- **Ch 17**: fcntl 在第 17 章异步通知 I/O 模型中再次说明;非阻塞 I/O(MSG_DONTWAIT)是非阻塞模型的基础。
- **POSIX readv/writev**: 标准 scatter/gather I/O 接口,源自 `<sys/uio.h>`,不限于套接字,可用于任意文件描述符。
