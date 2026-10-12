# Chapter 6: 基于UDP的服务器端/客户端

## Core Idea
UDP是无连接的、面向消息的传输方式,每次发送数据需附带目标地址(sendto),每次接收可同时获取发送端地址(recvfrom);一个UDP套接字可与多个主机通信,客户端首次调用sendto时自动分配IP和端口。

## Frameworks Introduced
- **UDP数据传输框架(sendto/recvfrom)**: UDP不保持连接状态,每次传输都需在sendto中指定目标地址,每次接收都通过recvfrom获取发送端地址信息。
  - When to use: UDP数据收发(无连接通信)
  - How: sendto(sock, data, len, 0, &dest_addr, addr_len)发送;recvfrom(sock, buf, len, 0, &from_addr, &addr_len)接收

- **UDP地址自动分配框架**: UDP客户端无需调用bind,首次调用sendto时操作系统自动给套接字分配IP和端口号,该地址一直保留到程序结束。
  - When to use: UDP客户端(也可显式bind,但通常不必要)
  - How: 创建套接字后直接sendto -> 操作系统首次调用时自动分配 -> 地址保留至程序结束

- **已连接UDP(Connected UDP)框架**: 对UDP套接字调用connect不会建立连接,而是注册目标地址信息,之后可用write/read替代sendto/recvfrom,提升性能。
  - When to use: 需要与同一目标主机频繁通信,或需要感知连接异常时
  - How: connect(udp_sock, ...)注册目标 -> write/read收发 -> 仅需调用一次connect

## Key Concepts
- **UDP(User Datagram Protocol)**: 无连接的、面向消息的传输层协议,不保证可靠性但速度快
- **sendto**: `ssize_t sendto(int sock, void *buff, size_t nbytes, int flags, struct sockaddr *to, socklen_t addrlen)`,向指定地址发送UDP数据
- **recvfrom**: `ssize_t recvfrom(int sock, void *buff, size_t nbytes, int flags, struct sockaddr *from, socklen_t *addrlen)`,接收UDP数据并获取发送端地址
- **面向消息(Message-oriented)**: UDP存在数据边界,每次sendto的数据作为一个独立消息,每次recvfrom接收一个完整消息
- **无连接(Connectionless)**: UDP不建立连接,每次传输独立,一个套接字可与多个主机通信
- **已连接UDP(Connected UDP)**: 对UDP套接字调用connect注册目标地址后的状态,可用write/read替代sendto/recvfrom
- **未连接UDP(Unconnected UDP)**: 默认状态,每次传输需指定目标地址

## Mental Models
- Use "sending a letter" when understanding UDP: each sendto is like mailing a letter with a destination address written on it; no prior connection needed, but delivery isn't guaranteed
- Use "one socket, many peers" when designing UDP servers: a single UDP socket can exchange data with multiple hosts, unlike TCP's one-to-one requirement
- Use "auto-bind on first sendto" when wondering about UDP client addressing: the OS assigns IP and port on the first sendto call, and keeps it for the program's lifetime
- Use "connect for UDP = register address, not handshake" when using connected UDP: it doesn't establish a connection, just lets you use write/read instead of sendto/recvfrom

## Anti-patterns
- **用TCP思维理解UDP数据边界**: UDP存在数据边界,调用3次sendto发送的数据必须调用3次recvfrom才能全部接收——与TCP的"无边界"相反
- **UDP客户端手动bind**: UDP客户端首次sendto时自动分配地址,通常无需额外bind(除非需要固定端口)
- **认为connect只能用于TCP**: UDP套接字也可调用connect,效果是注册目标地址而非建立连接

## Code Examples
```c
// uecho_server.c —— UDP回声服务器
int serv_sock;
char message[BUF_SIZE];
int str_len;
socklen_t clnt_adr_sz;
struct sockaddr_in serv_adr, clnt_adr;

serv_sock = socket(PF_INET, SOCK_DGRAM, 0);   // 创建UDP套接字(SOCK_DGRAM)

memset(&serv_adr, 0, sizeof(serv_adr));
serv_adr.sin_family = AF_INET;
serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
serv_adr.sin_port = htons(atoi(argv[1]));

bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr));

while (1)
{
    clnt_adr_sz = sizeof(clnt_adr);
    str_len = recvfrom(serv_sock, message, BUF_SIZE, 0,
                       (struct sockaddr*)&clnt_adr, &clnt_adr_sz);  // 接收并获取发送端地址
    sendto(serv_sock, message, str_len, 0,
           (struct sockaddr*)&clnt_adr, clnt_adr_sz);               // 向发送端回传
}
close(serv_sock);
```
- **What it demonstrates**: UDP服务器无需listen/accept,直接bind后循环recvfrom/sendto,一个套接字服务多个客户端

```c
// uecho_client.c —— UDP回声客户端(无connect,无bind)
int sock;
char message[BUF_SIZE];
int str_len;
socklen_t adr_sz;
struct sockaddr_in serv_adr, from_adr;

sock = socket(PF_INET, SOCK_DGRAM, 0);   // 创建UDP套接字

memset(&serv_adr, 0, sizeof(serv_adr));
serv_adr.sin_family = AF_INET;
serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
serv_adr.sin_port = htons(atoi(argv[2]));

while (1)
{
    fputs("Insert message(q to quit): ", stdout);
    fgets(message, sizeof(message), stdin);
    if (!strcmp(message, "q\n") || !strcmp(message, "Q\n"))
        break;

    sendto(sock, message, strlen(message), 0,
           (struct sockaddr*)&serv_adr, sizeof(serv_adr));  // 首次调用自动分配地址
    adr_sz = sizeof(from_adr);
    str_len = recvfrom(sock, message, BUF_SIZE, 0,
                       (struct sockaddr*)&from_adr, &adr_sz);
    message[str_len] = 0;
    printf("Message from server: %s", message);
}
close(sock);
```
- **What it demonstrates**: UDP客户端无需connect和bind,直接sendto/recvfrom收发数据

## Reference Tables

| 特性 | TCP(SOCK_STREAM) | UDP(SOCK_DGRAM) |
|------|-----------------|----------------|
| 连接状态 | 保持连接 | 无连接 |
| 数据边界 | 不存在 | 存在 |
| 可靠性 | 可靠(确认+重传) | 不可靠 |
| 数据收发函数 | write/read | sendto/recvfrom |
| 一对多 | 不支持(一一对应) | 支持(一个套接字多主机) |
| 地址分配 | 客户端connect时自动 | 首次sendto时自动 |
| listen/accept | 需要 | 不需要 |

| 函数 | 参数 | 说明 |
|------|------|------|
| sendto | sock, buff, nbytes, flags, to, addrlen | 传输数据+目标地址 |
| recvfrom | sock, buff, nbytes, flags, from, *addrlen | 接收数据+获取发送端地址 |

| UDP地址分配方式 | 说明 |
|----------------|------|
| 自动分配 | 首次sendto时OS自动分配IP和端口,保留至程序结束 |
| 显式bind | 手动调用bind分配(与TCP相同,bind不区分TCP/UDP) |

## Key Takeaways
1. UDP是无连接的面向消息协议,每次sendto需指定目标地址,每次recvfrom可获取发送端地址
2. UDP存在数据边界,3次sendto必须对应3次recvfrom——与TCP无边界特性相反
3. 一个UDP套接字可与多个主机通信,无需为每个客户端创建新套接字
4. UDP客户端首次调用sendto时操作系统自动分配IP和端口号,通常无需额外bind
5. 对UDP套接字调用connect不会建立连接,而是注册目标地址(已连接UDP),之后可用write/read替代sendto/recvfrom

## Connects To
- **Ch 2**: UDP使用SOCK_DGRAM套接字类型,与TCP的SOCK_STREAM对比(第2章)
- **Ch 4**: 本章与第4章的TCP服务器/客户端形成对比,展示了无连接通信模型
- **Ch 7**: UDP无需优雅断开(无连接),但TCP的半关闭问题在第7章讨论
- **Ch 8**: UDP也可使用域名获取地址(gethostbyname),与TCP相同
