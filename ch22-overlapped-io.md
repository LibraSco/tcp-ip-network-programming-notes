# Chapter 22: 重叠I/O模型

## Core Idea
重叠I/O（Overlapped I/O）是Windows平台真正的异步I/O模型：通过WSASocket创建具有WSA_FLAG_OVERLAPPED属性的套接字，使用WSASend/WSARecv发起非阻塞I/O后立即返回；I/O完成状态通过两种方式确认——基于事件对象（WSAOVERLAPPED.hEvent）或基于Completion Routine回调函数（需线程进入alertable wait状态）。

## Frameworks Introduced
- **重叠I/O（Overlapped I/O）**: 非阻塞模式I/O加上I/O完成状态确认机制的完整模型
  - When to use: 需要异步执行I/O操作并在完成后获取结果的Windows高性能服务器
  - How: WSASocket创建重叠套接字 -> WSASend/WSARecv发起异步I/O -> 事件对象或Completion Routine确认完成

- **基于事件对象的I/O完成确认**: 通过WSAOVERLAPPED结构体的hEvent成员确认I/O完成
  - When to use: 需要等待特定I/O操作完成时
  - How: WSACreateEvent创建事件 -> 填入overlapped.hEvent -> WSASend/WSARecv -> WSAWaitForMultipleEvents等待 -> WSAGetOverlappedResult获取结果

- **基于Completion Routine的I/O完成确认**: I/O完成时操作系统自动调用的回调函数
  - When to use: 不想手动等待事件，希望I/O完成时自动处理时
  - How: WSASend/WSARecv时传入CR函数地址 -> 调用SleepEx等进入alertable wait -> OS调用CR函数

## Key Concepts
- **WSAOVERLAPPED**: 重叠I/O的核心结构体，hEvent成员关联事件对象或自定义数据
- **WSA_FLAG_OVERLAPPED**: 赋予套接字重叠I/O属性的标志
- **WSASend/WSARecv**: 重叠I/O的数据收发函数，返回SOCKET_ERROR且WSAGetLastError为WSA_IO_PENDING表示I/O正在进行
- **WSA_IO_PENDING**: I/O尚未完成的错误码，非真正错误
- **WSAGetOverlappedResult**: 获取重叠I/O完成结果（传输字节数）的函数
- **Completion Routine**: I/O完成时自动调用的回调函数
- **alertable wait状态**: 线程等待接收OS消息的状态，是Completion Routine被调用的前提
- **PER_IO_DATA**: 封装套接字句柄、缓冲和WSABUF的自定义结构体，通过hEvent传递给CR

## Mental Models
- Use WSASocket with WSA_FLAG_OVERLAPPED when you need non-blocking async I/O on Windows
- Use WSA_IO_PENDING check after WSASend/WSARecv returns SOCKET_ERROR to distinguish pending from real errors
- Use event objects (hEvent) when you want to explicitly wait for I/O completion
- Use Completion Routine when you want automatic callback on I/O completion (requires alertable wait)
- Use WSAOVERLAPPED.hEvent to pass PER_IO_DATA address to Completion Routine (avoids needing a real event object)

## Anti-patterns
- **向lpOverlapped传递NULL**: 套接字将以阻塞模式工作，失去重叠I/O的意义
- **对同一套接字多个并发I/O使用同一个WSAOVERLAPPED变量**: 每个并发的重叠I/O操作需要独立的WSAOVERLAPPED结构体
- **Completion Routine方式下不进入alertable wait状态**: CR函数永远不会被调用，I/O完成通知丢失
- **WSARecv返回非SOCKET_ERROR时仍调用WSAGetOverlappedResult**: 返回0表示I/O已同步完成，此时recvBytes已有效

## Code Examples
```c
/* 基于事件对象的重叠I/O发送示例 */
WSAEVENT evObj;
WSAOVERLAPPED overlapped;
WSABUF dataBuf;
char msg[] = "Network is Computer!";
int sendBytes = 0;

/* 创建事件对象并初始化WSAOVERLAPPED */
evObj = WSACreateEvent();
memset(&overlapped, 0, sizeof(overlapped));
overlapped.hEvent = evObj;

dataBuf.len = strlen(msg) + 1;
dataBuf.buf = msg;

/* 发起异步发送 */
if (WSASend(hSocket, &dataBuf, 1, &sendBytes, 0, &overlapped, NULL)
    == SOCKET_ERROR)
{
    if (WSAGetLastError() == WSA_IO_PENDING)  /* I/O尚未完成 */
    {
        puts("Background data send");
        /* 等待I/O完成 */
        WSAWaitForMultipleEvents(1, &evObj, TRUE, WSA_INFINITE, FALSE);
        /* 获取实际传输结果 */
        WSAGetOverlappedResult(hSocket, &overlapped, &sendBytes, FALSE, NULL);
    }
}
printf("Send data size: %d \n", sendBytes);
WSACloseEvent(evObj);

/* --- Completion Routine方式 --- */
/* WSARecv传入CR函数地址，hEvent存储PER_IO_DATA地址 */
/* PER_IO_DATA结构体封装套接字句柄和缓冲信息 */
typedef struct {
    SOCKET hClntSock;
    char buf[BUF_SIZE];
    WSABUF wsaBuf;
} PER_IO_DATA;

/* Completion Routine函数原型（固定格式） */
void CALLBACK CompRoutine(DWORD dwError, DWORD szRecvBytes,
    LPWSAOVERLAPPED lpOverlapped, DWORD flags)
{
    /* 通过hEvent获取PER_IO_DATA地址 */
    LPPER_IO_DATA hbInfo = (LPPER_IO_DATA)(lpOverlapped->hEvent);
    SOCKET hSock = hbInfo->hClntSock;
    /* 处理接收到的数据... */
}

/* main线程需进入alertable wait状态使CR被调用 */
WSARecv(hRecvSock, &dataBuf, 1, &recvBytes, &flags, &overlapped, CompRoutine);
/* fAlertable=TRUE使线程进入alertable wait */
idx = WSAWaitForMultipleEvents(1, &evObj, FALSE, WSA_INFINITE, TRUE);
if (idx == WAIT_IO_COMPLETION)
    puts("Overlapped I/O Completed");
```
- **What it demonstrates**: 两种重叠I/O完成确认方式——事件对象方式（显式等待+获取结果）和Completion Routine方式（回调+alertable wait）

## Reference Tables
| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| WSASocket(af, type, proto, info, g, flags) | <winsock2.h> | 创建重叠I/O套接字 | 成功句柄，失败INVALID_SOCKET |
| WSASend(SOCKET, WSABUF*, count, sent*, flags, overlapped, CR) | <winsock2.h> | 异步发送数据 | 成功0，失败SOCKET_ERROR |
| WSARecv(SOCKET, WSABUF*, count, recvd*, flags, overlapped, CR) | <winsock2.h> | 异步接收数据 | 成功0，失败SOCKET_ERROR |
| WSAGetOverlappedResult(SOCKET, overlapped, transfer*, wait, flags*) | <winsock2.h> | 获取I/O完成结果 | 成功TRUE，失败FALSE |
| WSAGetLastError(void) | <winsock2.h> | 获取错误代码 | 错误码 |

| WSAOVERLAPPED结构体成员 | 类型 | 用途 |
|------------------------|------|------|
| Internal | DWORD | 系统内部使用 |
| InternalHigh | DWORD | 系统内部使用 |
| Offset | DWORD | 特殊用途 |
| OffsetHigh | DWORD | 特殊用途 |
| hEvent | WSAEVENT | 关联事件对象或自定义数据指针 |

| I/O完成确认方式 | 机制 | 需要事件对象 | 需要alertable wait | 适用场景 |
|----------------|------|-------------|-------------------|---------|
| 事件对象 | WSAOVERLAPPED.hEvent | 是 | 否 | 显式等待特定I/O完成 |
| Completion Routine | 回调函数 | 否(Dummy) | 是 | 自动处理I/O完成 |

| WSASend/WSARecv返回值 | 含义 | 处理方式 |
|----------------------|------|---------|
| 0 | I/O同步完成 | 传输字节数参数已有效 |
| SOCKET_ERROR + WSA_IO_PENDING | I/O异步进行中 | 需等待完成并获取结果 |
| SOCKET_ERROR + 其他错误码 | 真正错误 | 错误处理 |

## Key Takeaways
1. 重叠I/O = 非阻塞I/O + I/O完成确认机制，是Windows平台真正的异步I/O
2. WSASocket需传入WSA_FLAG_OVERLAPPED标志创建支持重叠I/O的套接字
3. WSASend/WSARecv返回SOCKET_ERROR时需检查WSAGetLastError，WSA_IO_PENDING表示I/O正在进行而非错误
4. 事件对象方式：hEvent关联事件 -> 等待signaled -> WSAGetOverlappedResult获取结果
5. Completion Routine方式：传入回调函数地址 -> 线程进入alertable wait -> OS调用CR函数
6. PER_IO_DATA结构体通过hEvent成员传递给CR函数，是连接I/O发起和完成处理的桥梁
7. Completion Routine的函数原型固定：void CALLBACK func(DWORD, DWORD, LPWSAOVERLAPPED, DWORD)

## Connects To
- **Ch 21**: 异步通知I/O处理的是"通知"而非真正的I/O，重叠I/O才是真正的异步I/O处理
- **Ch 23**: IOCP建立在重叠I/O基础上，用Completion Port替代事件对象和Completion Routine来确认I/O完成
- **Ch 17**: Linux的epoll配合非阻塞套接字可实现类似重叠I/O的异步处理效果
