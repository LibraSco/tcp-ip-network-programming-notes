# Chapter 4: 基于TCP的服务器端/客户端(1)

## Core Idea
TCP/IP协议栈分4层(链路层、IP层、TCP/UDP层、应用层),TCP在不可靠的IP层之上通过确认和重传机制赋予可靠性;TCP服务器端遵循socket->bind->listen->accept->read/write->close流程,客户端遵循socket->connect->read/write->close流程,回声服务器是迭代服务器的典型示例。

## Frameworks Introduced
- **TCP/IP四层协议栈框架**: 数据收发分为链路层(物理连接)、IP层(路径选择,面向消息不可靠)、TCP/UDP层(传输层,实际数据传输)、应用层(程序员定义的规则)。
  - When to use: 理解网络数据传输的分层处理过程
  - How: 链路层负责物理连接标准 -> IP层选择传输路径 -> TCP层保证可靠传输 -> 应用层定义数据交换规则

- **TCP三次握手框架**: TCP连接建立通过SYN/ACK三次消息交换完成,确保双方都准备好后再传输数据。
  - When to use: 理解connect和accept背后的连接建立过程
  - How: 客户端发SYN -> 服务器回SYN+ACK -> 客户端发ACK,连接建立

- **迭代服务器框架**: 服务器端循环调用accept受理一个客户端,完成数据交换后close,再受理下一个客户端。
  - When to use: 同时只需服务一个客户端的简单场景
  - How: while循环内 accept -> read/write数据交换 -> close客户端套接字

## Key Concepts
- **TCP/IP协议栈**: 分4层的层次化协议结构,化整为零解决"有效数据传输"问题
- **链路层**: 物理连接领域的标准化,定义LAN/WAN/MAN等网络标准
- **IP层**: 负责数据传输中的路径选择,是面向消息的、不可靠的协议
- **TCP层(传输层)**: 以IP层提供的路径为基础完成实际数据传输,通过确认和重传保证可靠性
- **应用层**: 程序员定义的服务器端与客户端间的数据传输规则(协议)
- **三次握手(Three-way Handshake)**: TCP建立连接的过程,SYN->SYN+ACK->ACK
- **连接请求等待队列**: 调用listen后生成的队列,存储客户端的连接请求,accept从中取请求
- **迭代服务器**: 一次只服务一个客户端,完成后再服务下一个的服务器模型
- **回声服务器(Echo Server)**: 将客户端发来的数据原样返回的服务器

## Mental Models
- Use "TCP adds reliability on top of unreliable IP" when understanding why TCP exists: IP only does path selection and may lose/reorder packets; TCP confirms receipt and retransmits lost data
- Use "listen creates a waiting queue, accept pops from it" when reasoning about server connection handling: connect doesn't mean accept was called, it means the request was recorded in the queue
- Use "client IP and port are auto-assigned at connect time by the OS kernel" when wondering why clients don't call bind
- Use "iterative server = one at a time" when designing simple servers that don't need concurrency

## Anti-patterns
- **connect成功不等于accept已调用**: connect返回只表示连接请求被记录到等待队列,并不表示服务器端已调用accept完成连接,connect返回后不一定立即进行数据交换
- **迭代服务器用于高并发场景**: 迭代服务器一次只能服务一个客户端,无法同时处理多个客户端连接,高并发需要多进程/IO复用(后续章节)
- **客户端先调用bind**: 客户端不需要手动bind,调用connect时操作系统自动分配IP和随机端口号

## Code Examples
```c
// echo_server.c —— 回声服务器(迭代)
int serv_sock, clnt_sock;
struct sockaddr_in serv_adr, clnt_adr;
socklen_t clnt_adr_sz;
char message[BUF_SIZE];

serv_sock = socket(PF_INET, SOCK_STREAM, 0);

memset(&serv_adr, 0, sizeof(serv_adr));
serv_adr.sin_family = AF_INET;
serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
serv_adr.sin_port = htons(atoi(argv[1]));

bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));
listen(serv_sock, 5);

for (i = 0; i < 5; i++)   // 迭代服务5个客户端
{
    clnt_adr_sz = sizeof(clnt_adr);
    clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);

    str_len = read(clnt_sock, message, BUF_SIZE);   // 读取数据
    write(clnt_sock, message, str_len);              // 原样返回(回声)

    close(clnt_sock);                                // 关闭客户端连接
}
close(serv_sock);
```
- **What it demonstrates**: TCP迭代服务器的基本流程——accept、read/write、close循环

```c
// echo_client.c —— 回声客户端
int sock;
struct sockaddr_in serv_adr;
char message[BUF_SIZE];
int str_len;

sock = socket(PF_INET, SOCK_STREAM, 0);

memset(&serv_adr, 0, sizeof(serv_adr));
serv_adr.sin_family = AF_INET;
serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
serv_adr.sin_port = htons(atoi(argv[2]));

connect(sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));  // 请求连接

while (1)
{
    fputs("Input message(Q to quit): ", stdout);
    fgets(message, BUF_SIZE, stdin);
    if (!strcmp(message, "q\n") || !strcmp(message, "Q\n"))
        break;

    write(sock, message, strlen(message));     // 发送数据
    str_len = read(sock, message, BUF_SIZE - 1);  // 接收回声
    message[str_len] = 0;
    printf("Message from server: %s", message);
}
close(sock);
```
- **What it demonstrates**: TCP客户端连接、数据收发、断开连接的完整流程

## Reference Tables

| 协议栈层 | 职责 | 特点 |
|---------|------|------|
| 链路层 | 物理连接标准化 | 定义LAN/WAN/MAN标准 |
| IP层 | 路径选择 | 面向消息、不可靠 |
| TCP/UDP层 | 实际数据传输 | TCP可靠,UDP简单 |
| 应用层 | 数据交换规则 | 程序员定义 |

| 步骤 | 服务器端 | 客户端 |
|------|---------|--------|
| 1 | socket() 创建套接字 | socket() 创建套接字 |
| 2 | bind() 分配地址 | (自动分配地址) |
| 3 | listen() 转为接收状态 | - |
| 4 | accept() 受理连接 | connect() 请求连接 |
| 5 | read()/write() 数据交换 | write()/read() 数据交换 |
| 6 | close() 断开连接 | close() 断开连接 |

| 三次握手步骤 | 方向 | 消息 |
|-------------|------|------|
| 1 | 客户端 -> 服务器 | SYN(SEQ=x) |
| 2 | 服务器 -> 客户端 | SYN+ACK(SEQ=y, ACK=x+1) |
| 3 | 客户端 -> 服务器 | ACK(ACK=y+1) |

## Key Takeaways
1. TCP/IP协议栈分4层,TCP在不可靠的IP层之上通过确认后传输和重传机制赋予可靠性
2. 服务器端套接字流程:socket->bind->listen->accept->read/write->close;客户端只需socket->connect->read/write->close
3. listen函数创建连接请求等待队列,connect只是将请求放入队列,accept才从队列取出并建立连接
4. 客户端的IP和端口在调用connect时由操作系统内核自动分配,无需手动bind
5. 迭代服务器通过for/while循环依次服务每个客户端,适用于单客户端场景
6. 回声服务器/客户端是理解TCP数据收发的最基本示例

## Connects To
- **Ch 1**: 本章回顾了第1章的hello_server.c,扩展为回声服务器
- **Ch 2**: TCP使用SOCK_STREAM套接字,UDP使用SOCK_DGRAM(第6章)
- **Ch 3**: 地址初始化基于sockaddr_in结构体和INADDR_ANY
- **Ch 5**: 回声客户端的"不完美实现"问题及TCP理论补充在第5章讲解
- **Ch 9**: TCP的Time-wait状态和套接字选项在第9章深入讨论
