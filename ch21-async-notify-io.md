# Chapter 21: 异步通知I/O模型

## Core Idea
异步通知I/O模型将"指定I/O监视对象"与"验证I/O状态变化"分离：WSAEventSelect注册套接字与事件类型的关联后立即返回，线程可执行其他任务；之后通过WSAWaitForMultipleEvents异步验证事件是否发生，再通过WSAEnumNetworkEvents区分具体事件类型。

## Frameworks Introduced
- **异步通知I/O（Async Notification I/O）**: 函数返回时刻与I/O状态变化时刻不一致的通知模型
  - When to use: 需要非阻塞方式监视多个套接字I/O状态的Windows服务器
  - How: WSAEventSelect注册监视 -> WSAWaitForMultipleEvents验证 -> WSAEnumNetworkEvents区分类型

- **WSAEventSelect模型**: select的异步版本，将套接字与事件对象关联
  - When to use: 替代select实现高性能Windows服务器I/O管理
  - How: 创建事件对象 -> WSAEventSelect关联套接字和事件 -> 等待事件对象signaled -> 枚举网络事件

## Key Concepts
- **同步I/O**: 函数调用返回时刻与I/O完成时刻一致（调用send等待传输完成才返回）
- **异步I/O**: 函数调用返回时刻与I/O完成时刻不一致（调用后立即返回，I/O后台进行）
- **通知I/O（Notification I/O）**: 通知I/O相关事件（如数据到达、可发送）的发生
- **WSAEventSelect**: 将套接字与事件对象关联，发生指定I/O事件时事件对象变signaled
- **WSACreateEvent**: 创建manual-reset模式、non-signaled状态的事件对象
- **WSAWaitForMultipleEvents**: 等待事件对象变为signaled，类似WaitForMultipleObjects但支持alertable wait
- **WSAEnumNetworkEvents**: 获取套接字上发生的具体网络事件类型
- **FD_READ/FD_ACCEPT/FD_CLOSE**: I/O事件类型常量

## Mental Models
- Use WSAEventSelect when you want to register a socket for I/O events without blocking
- Use WSAWaitForMultipleEvents when you want to asynchronously check if I/O events occurred
- Use manual-reset event objects when you need to check multiple signaled events in one pass
- Use WSAEnumNetworkEvents after WSAWaitForMultipleEvents returns to identify which specific event occurred

## Anti-patterns
- **使用auto-reset模式事件对象**: WSAWaitForMultipleEvents每次只能获取第一个signaled事件的索引，manual-reset模式允许逐个检查所有事件
- **调用WSAEventSelect后不再验证事件**: WSAEventSelect只是注册关联，不会阻塞等待事件发生
- **超过64个事件对象时不分组**: WSAWaitForMultipleEvents最多支持64个事件对象，超出需创建额外线程

## Code Examples
```c
/* 异步通知I/O回声服务器核心逻辑 */
WSAEVENT newEvent;
WSANETWORKEVENTS netEvents;
SOCKET hSockArr[WSA_MAXIMUM_WAIT_EVENTS];  /* 套接字数组 */
WSAEVENT hEventArr[WSA_MAXIMUM_WAIT_EVENTS]; /* 事件对象数组 */
int numOfClntSock = 0;

/* 注册服务器套接字的FD_ACCEPT事件 */
newEvent = WSACreateEvent();
WSAEventSelect(hServSock, newEvent, FD_ACCEPT);
hSockArr[numOfClntSock] = hServSock;
hEventArr[numOfClntSock] = newEvent;
numOfClntSock++;

while (1)
{
    /* 异步等待事件发生 - 返回signaled事件的索引 */
    posInfo = WSAWaitForMultipleEvents(
        numOfClntSock, hEventArr, FALSE, WSA_INFINITE, FALSE);
    startIdx = posInfo - WSA_WAIT_EVENT_0;

    /* 从startIdx开始逐个检查所有事件对象 */
    for (i = startIdx; i < numOfClntSock; i++)
    {
        sigEventIdx = WSAWaitForMultipleEvents(1, &hEventArr[i], TRUE, 0, FALSE);
        if (sigEventIdx == WSA_WAIT_FAILED || sigEventIdx == WSA_WAIT_TIMEOUT)
            continue;

        /* 获取具体发生的网络事件类型 */
        WSAEnumNetworkEvents(hSockArr[i], hEventArr[i], &netEvents);

        if (netEvents.lNetworkEvents & FD_ACCEPT) {  /* 新连接 */
            hClntSock = accept(hSockArr[i], (SOCKADDR*)&clntAdr, &clntAdrLen);
            newEvent = WSACreateEvent();
            WSAEventSelect(hClntSock, newEvent, FD_READ | FD_CLOSE);
            hSockArr[numOfClntSock] = hClntSock;
            hEventArr[numOfClntSock] = newEvent;
            numOfClntSock++;
        }
        if (netEvents.lNetworkEvents & FD_READ) {  /* 数据到达 */
            strLen = recv(hSockArr[i], msg, sizeof(msg), 0);
            send(hSockArr[i], msg, strLen, 0);
        }
        if (netEvents.lNetworkEvents & FD_CLOSE) {  /* 连接关闭 */
            WSACloseEvent(hEventArr[i]);
            closesocket(hSockArr[i]);
            numOfClntSock--;
            CompressSockets(hSockArr, i, numOfClntSock);
            CompressEvents(hEventArr, i, numOfClntSock);
        }
    }
}
```
- **What it demonstrates**: 异步通知I/O服务器的完整工作流程——注册事件、异步等待、枚举事件类型、处理不同I/O事件

## Reference Tables
| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| WSAEventSelect(SOCKET, WSAEVENT, long) | <winsock2.h> | 关联套接字与事件对象 | 成功0，失败SOCKET_ERROR |
| WSACreateEvent(void) | <winsock2.h> | 创建manual-reset事件对象 | 成功句柄，失败WSA_INVALID_EVENT |
| WSAWaitForMultipleEvents(...) | <winsock2.h> | 等待事件对象signaled | 事件索引/超时/失败 |
| WSAEnumNetworkEvents(SOCKET, WSAEVENT, LPWSANETWORKEVENTS) | <winsock2.h> | 获取发生的网络事件类型 | 成功0，失败SOCKET_ERROR |
| WSACloseEvent(WSAEVENT) | <winsock2.h> | 销毁事件对象 | 成功TRUE，失败FALSE |

| 事件类型 | 触发条件 |
|---------|---------|
| FD_READ | 输入缓冲收到数据，需要读取 |
| FD_WRITE | 可以非阻塞方式传输数据 |
| FD_OOB | 收到带外数据 |
| FD_ACCEPT | 有新的连接请求 |
| FD_CLOSE | 有断开连接的请求 |

| 模型特性 | select | WSAEventSelect |
|---------|--------|---------------|
| 工作方式 | 同步通知 | 异步通知 |
| 注册方式 | 每次调用需重新注册 | 一次注册持续有效 |
| 事件对象 | 文件描述符集合 | 事件对象(manual-reset) |
| 最大监视数 | FD_SETSIZE | WSA_MAXIMUM_WAIT_EVENTS(64) |
| 返回方式 | 需遍历全部检查 | 返回signaled事件索引 |

## Key Takeaways
1. 异步通知I/O将"监视对象注册"与"状态验证"分离，注册后线程可执行其他任务
2. WSAEventSelect将套接字与manual-reset事件对象关联，发生指定事件时事件对象变signaled
3. WSAWaitForMultipleEvents最多监视64个事件对象，返回第一个signaled事件的索引
4. 利用manual-reset特性，可通过循环逐个检查所有事件对象是否signaled
5. WSAEnumNetworkEvents同时获取事件类型并将事件对象重置为non-signaled
6. 套接字和事件对象必须在数组中保持对应关系（同索引），断开时需同时移除

## Connects To
- **Ch 12**: select是同步通知I/O模型，本章是其异步版本
- **Ch 17**: epoll的注册一次持续有效特性与WSAEventSelect类似
- **Ch 22**: 异步通知I/O是异步I/O的前置概念，重叠I/O是真正的异步I/O处理
- **Ch 20**: 事件对象（Event）的概念在第20章已介绍，本章使用manual-reset模式事件对象
