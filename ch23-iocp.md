# Chapter 23: IOCP

## Core Idea
IOCP（Input Output Completion Port）是Windows性能最优的I/O模型：创建Completion Port对象并将套接字与之关联，专职I/O线程通过GetQueuedCompletionStatus获取已完成的I/O信息；IOCP将非阻塞I/O、自动完成通知和线程池管理融为一体，以最少线程处理最大并发I/O。

## Frameworks Introduced
- **IOCP（Completion Port模型）**: 创建完成端口对象，将套接字I/O完成信息注册到CP对象，由专职线程获取并处理
  - When to use: Windows平台下需要处理大量并发连接的高性能服务器
  - How: CreateIoCompletionPort创建CP -> 关联套接字到CP -> 创建I/O线程 -> GetQueuedCompletionStatus获取完成I/O

- **CreateIoCompletionPort双重功能**: 同一函数实现两种功能——创建CP对象和关联套接字到CP
  - When to use: 第一次调用创建CP对象，后续调用将客户端套接字关联到CP
  - How: 创建时传INVALID_HANDLE_VALUE/NULL/0；关联时传套接字/CP句柄/CompletionKey

- **PER_HANDLE_DATA / PER_IO_DATA结构体**: IOCP中传递客户端信息和I/O信息的自定义数据结构
  - When to use: 需要在I/O完成时获取套接字信息和缓冲信息时
  - How: PER_HANDLE_DATA通过CompletionKey传递，PER_IO_DATA通过OVERLAPPED地址传递

## Key Concepts
- **Completion Port（CP）**: 注册已完成I/O信息的内核对象
- **CreateIoCompletionPort**: 创建CP对象或关联套接字到CP的双重功能函数
- **GetQueuedCompletionStatus**: 从CP对象获取已完成I/O信息的阻塞函数
- **CompletionKey**: 关联套接字时传入的自定义数据（通常为PER_HANDLE_DATA地址）
- **PER_HANDLE_DATA**: 封装套接字句柄和客户端地址的结构体
- **PER_IO_DATA**: 封装OVERLAPPED、WSABUF和缓冲的结构体（地址与overlapped成员地址相同）
- **NumberOfConcurrentThreads**: CP对象允许同时运行的I/O处理线程数上限
- **rwMode**: 区分I/O是读操作还是写操作的自定义标志（IOCP本身不区分读写完成）

## Mental Models
- Use CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0) to create a CP object
- Use CreateIoCompletionPort(socket, cpHandle, key, 0) to connect a socket to the CP
- Use GetQueuedCompletionStatus to block and wait for completed I/O in worker threads
- Use PER_HANDLE_DATA via CompletionKey to identify which client's I/O completed
- Use PER_IO_DATA (via OVERLAPPED pointer) to access buffer and distinguish read vs write completion
- Use rwMode field in PER_IO_DATA since IOCP does not distinguish read from write completion

## Anti-patterns
- **用纯重叠I/O（Completion Routine）实现服务器**: 需要交替调用非阻塞accept和SleepEx进入alertable wait，性能低下
- **创建过多或过少的I/O线程**: 线程数应与CPU核心数相同（GetSystemInfo获取），过多导致上下文切换开销
- **IOCP中不区分读写完成**: IOCP只通知I/O完成，不区分是输入还是输出，需通过rwMode自定义标志区分
- **PER_IO_DATA中OVERLAPPED不是第一个成员**: 结构体变量地址必须与第一个成员地址相同，否则无法通过OVERLAPPED指针获取PER_IO_DATA

## Code Examples
```c
/* IOCP回声服务器完整实现 */

/* 数据结构定义 */
typedef struct {              // socket info (PER_HANDLE_DATA)
    SOCKET hClntSock;
    SOCKADDR_IN clntAdr;
} PER_HANDLE_DATA, *LPPER_HANDLE_DATA;

typedef struct {              // buffer info (PER_IO_DATA)
    OVERLAPPED overlapped;   // 必须是第一个成员！
    WSABUF wsaBuf;
    char buffer[BUF_SIZE];
    int rwMode;              // READ or WRITE
} PER_IO_DATA, *LPPER_IO_DATA;

/* main函数 - 创建CP、线程、关联套接字 */
int main(int argc, char* argv[])
{
    HANDLE hComPort;
    SYSTEM_INFO sysInfo;
    LPPER_IO_DATA ioInfo;
    LPPER_HANDLE_DATA handleInfo;

    /* 1. 创建Completion Port对象 */
    hComPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

    /* 2. 获取CPU核心数，创建相应数量的I/O线程 */
    GetSystemInfo(&sysInfo);
    for (i = 0; i < sysInfo.dwNumberOfProcessors; i++)
        _beginthreadex(NULL, 0, EchoThreadMain, (LPVOID)hComPort, 0, NULL);

    /* 3. 创建服务器套接字... bind... listen... */

    while (1)
    {
        hClntSock = accept(hServSock, (SOCKADDR*)&clntAdr, &addrLen);

        /* 4. 分配PER_HANDLE_DATA，保存客户端信息 */
        handleInfo = (LPPER_HANDLE_DATA)malloc(sizeof(PER_HANDLE_DATA));
        handleInfo->hClntSock = hClntSock;
        memcpy(&(handleInfo->clntAdr), &clntAdr, addrLen);

        /* 5. 关联客户端套接字到CP对象 */
        CreateIoCompletionPort((HANDLE)hClntSock, hComPort, (DWORD)handleInfo, 0);

        /* 6. 分配PER_IO_DATA，发起首次异步接收 */
        ioInfo = (LPPER_IO_DATA)malloc(sizeof(PER_IO_DATA));
        memset(&(ioInfo->overlapped), 0, sizeof(OVERLAPPED));
        ioInfo->wsaBuf.len = BUF_SIZE;
        ioInfo->wsaBuf.buf = ioInfo->buffer;
        ioInfo->rwMode = READ;
        WSARecv(handleInfo->hClntSock, &(ioInfo->wsaBuf),
            1, &recvBytes, &flags, &(ioInfo->overlapped), NULL);
    }
    return 0;
}

/* I/O线程函数 - 获取并处理已完成的I/O */
DWORD WINAPI EchoThreadMain(LPVOID pComPort)
{
    HANDLE hComPort = (HANDLE)pComPort;
    SOCKET sock;
    DWORD bytesTrans;
    LPPER_HANDLE_DATA handleInfo;
    LPPER_IO_DATA ioInfo;
    DWORD flags = 0;

    while (1)
    {
        /* 获取已完成的I/O信息（阻塞直到有I/O完成） */
        GetQueuedCompletionStatus(hComPort, &bytesTrans,
            (LPDWORD)&handleInfo, (LPOVERLAPPED*)&ioInfo, INFINITE);

        sock = handleInfo->hClntSock;

        if (ioInfo->rwMode == READ)    /* 读取完成 */
        {
            if (bytesTrans == 0) {     /* EOF，客户端断开 */
                closesocket(sock);
                free(handleInfo);
                free(ioInfo);
                continue;
            }
            /* 发送回声数据 */
            memset(&(ioInfo->overlapped), 0, sizeof(OVERLAPPED));
            ioInfo->wsaBuf.len = bytesTrans;
            ioInfo->rwMode = WRITE;
            WSASend(sock, &(ioInfo->wsaBuf), 1, NULL, 0, &(ioInfo->overlapped), NULL);

            /* 同时发起新的接收 */
            ioInfo = (LPPER_IO_DATA)malloc(sizeof(PER_IO_DATA));
            memset(&(ioInfo->overlapped), 0, sizeof(OVERLAPPED));
            ioInfo->wsaBuf.len = BUF_SIZE;
            ioInfo->wsaBuf.buf = ioInfo->buffer;
            ioInfo->rwMode = READ;
            WSARecv(sock, &(ioInfo->wsaBuf), 1, NULL, &flags, &(ioInfo->overlapped), NULL);
        }
        else    /* 写入完成 */
        {
            free(ioInfo);    /* 释放写操作使用的PER_IO_DATA */
        }
    }
    return 0;
}
```
- **What it demonstrates**: IOCP服务器的完整架构——CP对象创建、套接字关联、线程池处理已完成的I/O、通过PER_HANDLE_DATA和PER_IO_DATA传递上下文信息

## Reference Tables
| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| CreateIoCompletionPort(FileHandle, ExistingCP, CompletionKey, NumThreads) | <windows.h> | 创建CP或关联套接字 | 成功CP句柄，失败NULL |
| GetQueuedCompletionStatus(CP, bytes*, key*, overlapped**, timeout) | <windows.h> | 获取已完成I/O | 成功TRUE，失败FALSE |
| GetSystemInfo(LPSYSTEM_INFO) | <windows.h> | 获取系统信息(含CPU数) | 无 |

| CreateIoCompletionPort用途 | FileHandle | ExistingCompletionPort | CompletionKey | NumberOfConcurrentThreads |
|---------------------------|------------|------------------------|---------------|--------------------------|
| 创建CP对象 | INVALID_HANDLE_VALUE | NULL | 0 | 最大线程数 |
| 关联套接字到CP | 套接字句柄 | CP对象句柄 | PER_HANDLE_DATA地址 | 忽略(传0) |

| GetQueuedCompletionStatus参数 | 获取的信息 | 来源 |
|-------------------------------|----------|------|
| lpNumberOfBytes | I/O传输的字节数 | 系统填充 |
| lpCompletionKey | CreateIoCompletionPort时传入的CompletionKey | 关联套接字时设置 |
| lpOverlapped | WSASend/WSARecv时传入的OVERLAPPED地址 | I/O发起时设置(=PER_IO_DATA地址) |

| 数据结构 | 内容 | 传递方式 |
|---------|------|---------|
| PER_HANDLE_DATA | 套接字句柄、客户端地址 | CompletionKey参数 |
| PER_IO_DATA | OVERLAPPED、WSABUF、缓冲、rwMode | OVERLAPPED地址(结构体首成员) |

| IOCP性能优势 | 说明 |
|-------------|------|
| 非阻塞I/O | 不会因I/O等待造成线程阻塞 |
| 无需循环查找 | GetQueuedCompletionStatus直接获取已完成I/O |
| 无需套接字数组管理 | CP对象自动管理关联的套接字 |
| 可调整线程数 | 按CPU核心数创建最优线程数 |

## Key Takeaways
1. IOCP = 非阻塞重叠I/O + Completion Port完成通知 + 线程池管理，是Windows最优I/O模型
2. CreateIoCompletionPort有双重功能：第一次调用创建CP对象，后续调用将套接字关联到CP
3. I/O线程数应与CPU核心数相同（通过GetSystemInfo获取），以最小线程数处理最大并发I/O
4. GetQueuedCompletionStatus阻塞等待I/O完成，返回时获取CompletionKey（套接字信息）和OVERLAPPED（I/O信息）
5. PER_IO_DATA的OVERLAPPED必须是第一个成员，使结构体地址与OVERLAPPED地址相同，从而通过OVERLAPPED指针访问整个PER_IO_DATA
6. IOCP不区分读写完成，需在PER_IO_DATA中自定义rwMode标志区分
7. 读取完成后可同时发起WSASend（回声）和新的WSARecv（继续接收），实现流水线处理

## Connects To
- **Ch 22**: IOCP建立在重叠I/O基础上，用Completion Port替代事件对象/Completion Routine确认I/O完成
- **Ch 17**: IOCP是Windows的终极I/O模型，与Linux的epoll形成跨平台高性能I/O对比
- **Ch 24**: HTTP服务器因连接短暂（1次请求即断开），不适合展示IOCP优势；需长连接频繁交互场景才能体现
