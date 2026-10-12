# Chapter 7: 优雅地断开套接字连接

## Core Idea
close/closesocket会同时断开输入和输出两个流,导致无法在发送EOF后继续接收对方数据;半关闭(Half-close)通过shutdown函数只断开其中一个流,既能向对方传递EOF表示数据发送完毕,又能保持接收能力——这是优雅断开连接的关键。

## Frameworks Introduced
- **流(Stream)分离框架**: 套接字连接建立后形成双向数据通道,包含输入流(接收)和输出流(发送)两个独立流;断开连接可选择只断一个流(半关闭)或同时断两个流(完全关闭)。
  - When to use: 需要在通知对方"我已发完数据"的同时继续接收对方数据时
  - How: 调用shutdown(sock, SHUT_WR)只断开输出流 -> 对方收到EOF -> 继续read接收对方数据 -> 最后close完全断开

- **shutdown半关闭框架**: 使用shutdown函数的howto参数指定断开方式——SHUT_RD断输入流、SHUT_WR断输出流、SHUT_RDWR同时断两个流。
  - When to use: 服务器传输文件后需通知客户端传输结束,同时接收客户端确认消息
  - How: 发完最后数据 -> shutdown(sock, SHUT_WR)发送EOF -> read接收对方回应 -> close

- **EOF传递框架**: 断开输出流时TCP自动向对方发送FIN消息(即EOF),对方read函数返回0表示收到EOF;close和shutdown(SHUT_WR)都能发EOF,但close同时断输入流导致无法继续接收。
  - When to use: 需要通知对方数据传输结束且不与数据内容冲突时
  - How: 用shutdown(SHUT_WR)而非close来发EOF -> 保留输入流继续接收

## Key Concepts
- **半关闭(Half-close)**: 只断开流的一部分(输入或输出),使一方可以传输但无法接收,或可以接收但无法传输
- **流(Stream)**: 套接字连接建立后可交换数据的状态,双向通信需要两个流(输入流+输出流)
- **shutdown函数**: `int shutdown(int sock, int howto)`,用于半关闭,只断开指定的流
- **SHUT_RD**: 断开输入流,套接字无法接收数据,缓冲数据被抹去
- **SHUT_WR**: 断开输出流,无法传输数据,但缓冲中未传输的数据会传完后再断开
- **SHUT_RDWR**: 同时断开输入输出流,相当于分2次调用shutdown(SHUT_RD和SHUT_WR)
- **close vs shutdown**: close完全断开两个流(无法再收发);shutdown可选断开一个流(可继续收或发)
- **EOF传递**: 断开输出流时向对方发送FIN(EOF),对方read返回0

## Mental Models
- Use "two independent streams" when thinking about TCP connection: one stream for sending, one for receiving; closing one doesn't have to close the other
- Use "shutdown(SHUT_WR) to signal 'I'm done sending but still listening'" when transferring files: send EOF to mark end, keep read open for the client's "Thank you"
- Use "close = nuclear option, shutdown = surgical option" when choosing disconnection method: close kills both streams; shutdown lets you keep one alive

## Anti-patterns
- **用close代替shutdown通知传输结束**: close同时断开输入输出流,发送EOF后无法再接收对方数据——文件服务器传输完文件后想接收客户端确认消息时,close会导致接收失败
- **用约定特殊字符表示传输结束**: 文件内容可能包含该特殊字符导致冲突——应使用EOF(通过断开输出流传递)通知结束
- **客户端无休止调用read等待文件结束**: 客户端无法知道何时文件传输完毕,可能导致程序阻塞——应通过服务器半关闭传递EOF让read返回0

## Code Examples
```c
// file_server.c —— 基于半关闭的文件传输服务器
int serv_sd, clnt_sd;
FILE *fp;
char buf[BUF_SIZE];
int read_cnt;

fp = fopen("file_server.c", "rb");
serv_sd = socket(PF_INET, SOCK_STREAM, 0);
// ... bind, listen, accept ...

while (1)
{
    read_cnt = fread((void*)buf, 1, BUF_SIZE, fp);
    if (read_cnt < BUF_SIZE)      // 文件读取完毕
    {
        write(clnt_sd, buf, read_cnt);  // 发送最后一段
        break;
    }
    write(clnt_sd, buf, BUF_SIZE);      // 发送完整缓冲
}

shutdown(clnt_sd, SHUT_WR);     // 半关闭:断开输出流,向客户端发送EOF
read(clnt_sd, buf, BUF_SIZE);   // 仍可接收客户端的确认消息
printf("Message from client: %s \n", buf);

fclose(fp);
close(clnt_sd); close(serv_sd);  // 完全关闭
```
- **What it demonstrates**: 服务器传输完文件后用shutdown(SHUT_WR)发送EOF,同时保留输入流接收客户端确认消息——close无法做到这一点

```c
// file_client.c —— 接收文件并回复确认
int sd;
FILE *fp;
char buf[BUF_SIZE];
int read_cnt;

sd = socket(PF_INET, SOCK_STREAM, 0);
// ... connect ...

fp = fopen("receive.dat", "wb");  // 打开文件用于写入
while ((read_cnt = read(sd, buf, BUF_SIZE)) != 0)  // read返回0表示收到EOF
    fwrite((void*)buf, 1, read_cnt, fp);

write(sd, "Thank you", 10);    // 文件接收完毕,发送确认
fclose(fp);
close(sd);
```
- **What it demonstrates**: 客户端循环read直到返回0(收到服务器EOF),然后发送"Thank you"确认

## Reference Tables

| 函数 | 断开范围 | 能否发EOF | 能否继续接收 | 能否继续发送 |
|------|---------|----------|------------|------------|
| close | 输入+输出 | 是(同时) | 否 | 否 |
| shutdown(SHUT_RD) | 仅输入 | 否 | 否(缓冲清空) | 是 |
| shutdown(SHUT_WR) | 仅输出 | 是 | 是 | 否 |
| shutdown(SHUT_RDWR) | 输入+输出 | 是 | 否 | 否 |

| howto参数 | 断开的流 | 效果 |
|-----------|---------|------|
| SHUT_RD | 输入流 | 无法接收数据,缓冲数据被抹去 |
| SHUT_WR | 输出流 | 无法发送数据,缓冲未传数据传完后断开,向对方发EOF |
| SHUT_RDWR | 输入+输出流 | 同时断开两个流(相当于SHUT_RD+SHUT_WR) |

## Key Takeaways
1. 套接字连接形成两个独立流(输入流+输出流),断开连接可以选择只断一个流(半关闭)或同时断两个流(完全关闭)
2. close/closesocket同时断开两个流,发送EOF后无法继续接收——不"优雅"
3. shutdown(sock, SHUT_WR)只断开输出流,既能向对方发送EOF表示数据发送完毕,又能保留输入流继续接收
4. 半关闭的典型场景:服务器传完文件后用shutdown(SHUT_WR)发EOF通知结束,同时read接收客户端的确认消息
5. 客户端循环read直到返回0(收到EOF)即可知道文件传输结束,避免了用特殊字符标记的冲突问题

## Connects To
- **Ch 5**: 第5章提到close传递EOF但同时关闭两个流的问题,本章通过半关闭解决
- **Ch 4**: TCP连接建立(三次握手)在第4章,本章关注连接断开(半关闭)过程
- **Ch 9**: 套接字断开后的Time-wait状态和SO_REUSEADDR在第9章讨论
- **TCP四次握手**: 半关闭基于TCP的FIN/ACK四次握手过程实现流的独立断开
