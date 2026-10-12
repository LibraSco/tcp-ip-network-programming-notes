# Chapter 1: 理解网络编程和套接字

## Core Idea
网络编程的本质是通过套接字(socket)完成数据交换;在Linux中,socket与文件一样通过文件描述符操作,服务器端的建立遵循"socket -> bind -> listen -> accept"四步调用顺序。

## Frameworks Introduced
- **套接字编程模型(电话机比喻)**: 将套接字创建过程类比为安装电话——socket创建电话机,bind分配号码,listen连接电话线,accept拿起话筒接听。
  - When to use: 理解服务器端套接字建立流程时
  - How: socket()创建套接字 -> bind()分配地址和端口 -> listen()转为接收状态 -> accept()受理连接

- **Linux文件I/O模型**: Linux不区分socket与文件,二者都通过文件描述符操作,因此文件I/O函数(read/write/close)同样适用于套接字。
  - When to use: Linux平台下的所有网络数据收发
  - How: open打开文件获取fd -> write/read收发数据 -> close关闭

## Key Concepts
- **套接字(Socket)**: 网络数据交换用的出口,是网络编程的接口
- **文件描述符(File Descriptor)**: 系统分配给文件或套接字的整数,用于标识和操作该对象
- **服务器端套接字**: 调用listen后转为可接收连接状态的套接字,又称监听(listening)套接字
- **客户端套接字**: 用于请求连接的套接字,创建后直接调用connect
- **底层文件访问(Low-Level File Access)**: 操作系统独立提供的(非ANSI标准)文件操作函数
- **socket函数**: `int socket(int domain, int type, int protocol)`,成功返回文件描述符,失败返回-1
- **bind函数**: `int bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen)`,给套接字分配IP和端口
- **listen函数**: `int listen(int sockfd, int backlog)`,将套接字转为可接收连接请求状态
- **accept函数**: `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`,受理连接请求,返回新套接字文件描述符
- **connect函数**: `int connect(int sockfd, struct sockaddr *serv_addr, socklen_t addrlen)`,客户端向服务器端发送连接请求

## Mental Models
- Use the "telephone installation" model when learning server socket setup: socket=买电话机, bind=分配号码, listen=连接电话线, accept=拿起话筒接听
- Use "file descriptor as integer handle" when reasoning about why read/write work on sockets in Linux: the OS returns an integer that becomes the communication channel between programmer and OS
- Use "server does 4 steps, client does 2 steps" when remembering the asymmetry: server needs socket+bind+listen+accept, client only needs socket+connect

## Anti-patterns
- **混淆服务器端与客户端创建过程**: socket()创建的套接字本身不区分服务器/客户端,只有后续调用bind+listen才成为服务器端,调用connect才成为客户端
- **忽略accept的阻塞特性**: 在没有连接请求时调用accept,函数不会返回,直到有连接请求为止——不要误以为是死循环

## Code Examples
```c
// hello_server.c —— 服务器端基本结构
int serv_sock;
int clnt_sock;
struct sockaddr_in serv_addr;
struct sockaddr_in clnt_addr;
socklen_t clnt_addr_size;
char message[] = "Hello World!";

serv_sock = socket(PF_INET, SOCK_STREAM, 0);       // 第一步:创建套接字
if (serv_sock == -1)
    error_handling("socket() error");

memset(&serv_addr, 0, sizeof(serv_addr));
serv_addr.sin_family = AF_INET;
serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
serv_addr.sin_port = htons(atoi(argv[1]));

if (bind(serv_sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1)  // 第二步:分配地址
    error_handling("bind() error");

if (listen(serv_sock, 5) == -1)                    // 第三步:转为接收状态
    error_handling("listen() error");

clnt_addr_size = sizeof(clnt_addr);
clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_addr_size);  // 第四步:受理连接
if (clnt_sock == -1)
    error_handling("accept() error");

write(clnt_sock, message, sizeof(message));       // 传输数据
close(clnt_sock);
close(serv_sock);
```
- **What it demonstrates**: 服务器端四步调用顺序(socket/bind/listen/accept)及数据传输

```c
// hello_client.c —— 客户端基本结构
int sock;
struct sockaddr_in serv_addr;
char message[30];
int str_len;

sock = socket(PF_INET, SOCK_STREAM, 0);            // 第一步:创建套接字
if (sock == -1)
    error_handling("socket() error");

memset(&serv_addr, 0, sizeof(serv_addr));
serv_addr.sin_family = AF_INET;
serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
serv_addr.sin_port = htons(atoi(argv[2]));

if (connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1)  // 第二步:请求连接
    error_handling("connect() error!");

str_len = read(sock, message, sizeof(message) - 1);
if (str_len == -1)
    error_handling("read() error!");

printf("Message from server : %s \n", message);
close(sock);
```
- **What it demonstrates**: 客户端两步调用(socket/connect)及数据接收

## Reference Tables

| 文件描述符 | 对象 |
|-----------|------|
| 0 | 标准输入 (Standard Input) |
| 1 | 标准输出 (Standard Output) |
| 2 | 标准错误 (Standard Error) |
| 3+ | 程序创建的文件/套接字(按序分配) |

| 文件打开模式 | 含义 |
|-------------|------|
| O_CREAT | 必要时创建文件 |
| O_TRUNC | 删除全部现有数据 |
| O_APPEND | 维持现有数据,保存到其后 |
| O_RDONLY | 只读打开 |
| O_WRONLY | 只写打开 |
| O_RDWR | 读写打开 |

| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| socket | sys/socket.h | 创建套接字 | 文件描述符 / -1 |
| bind | sys/socket.h | 分配地址信息 | 0 / -1 |
| listen | sys/socket.h | 转为接收连接状态 | 0 / -1 |
| accept | sys/socket.h | 受理连接请求 | 文件描述符 / -1 |
| connect | sys/socket.h | 请求连接 | 0 / -1 |
| open | fcntl.h | 打开文件 | 文件描述符 / -1 |
| read | unistd.h | 读取数据 | 接收字节数 / -1 |
| write | unistd.h | 写入数据 | 传输字节数 / -1 |
| close | unistd.h | 关闭文件/套接字 | 0 / -1 |

## Key Takeaways
1. 服务器端套接字建立严格遵循 socket -> bind -> listen -> accept 四步顺序,缺少任何一步都无法正常工作
2. 在Linux中socket即文件,read/write/close等文件I/O函数可直接用于套接字数据收发,这是Linux与Windows的关键区别
3. 文件描述符从3开始分配(0/1/2预留给标准I/O),它是程序与操作系统间通信的整数句柄
4. 客户端无需调用bind,调用connect时操作系统会自动分配IP和端口号
5. accept函数在没有连接请求时会阻塞(不返回),这是正常的等待行为而非错误

## Connects To
- **Ch 2**: socket函数的三个参数(domain/type/protocol)在第2章详细解释
- **Ch 3**: sockaddr_in结构体和地址初始化在第3章深入讲解
- **Ch 4**: 本章的hello_server/client在第4章扩展为回声(echo)服务器/客户端
- **POSIX标准**: 文件描述符和底层文件访问源自POSIX/UNIX标准
