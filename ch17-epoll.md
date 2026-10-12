# Chapter 17: 优于select的epoll

## Core Idea
epoll是Linux下高性能I/O多路复用机制，通过epoll_create/epoll_ctl/epoll_wait三个函数实现事件驱动的I/O管理；相比select的两大缺点（文件描述符数量限制和全量遍历），epoll仅在发生事件的套接字上返回，且支持条件触发（LT）和边缘触发（ET）两种模式。

## Frameworks Introduced
- **epoll**: Linux内核级别的事件通知I/O多路复用机制
  - When to use: 需要同时监视大量套接字的高并发服务器场景
  - How: epoll_create创建实例 -> epoll_ctl注册套接字 -> epoll_wait等待事件

- **条件触发（Level Trigger, LT）**: 只要输入缓冲有数据就持续通知的模式
  - When to use: 需要简化编程逻辑，确保不遗漏数据的场景
  - How: 默认模式，event.events = EPOLLIN

- **边缘触发（Edge Trigger, ET）**: 仅在状态变化时通知一次的模式
  - When to use: 需要极致性能，减少epoll_wait调用次数的场景
  - How: event.events = EPOLLIN | EPOLLET，需配合非阻塞套接字

## Key Concepts
- **epoll_create**: 创建epoll实例，返回epoll文件描述符
- **epoll_ctl**: 向epoll实例添加、修改或删除监视的文件描述符
- **epoll_wait**: 等待已注册的文件描述符上发生事件
- **struct epoll_event**: epoll事件结构体，包含events标志和data联合体
- **EPOLLIN**: 可读事件标志
- **EPOLLOUT**: 可写事件标志
- **EPOLLET**: 边缘触发模式标志
- **EPOLL_CTL_ADD/MOD/DEL**: epoll_ctl的操作类型

## Mental Models
- Use epoll instead of select when monitoring more than a few hundred sockets
- Use Level Trigger (default) when you want simple, safe code that won't miss data
- Use Edge Trigger when you need maximum performance and can handle non-blocking I/O properly
- Use EPOLLET with non-blocking sockets when implementing edge-triggered epoll to avoid blocking

## Anti-patterns
- **在边缘触发模式下使用阻塞套接字**: ET模式只通知一次，若数据未读完将导致数据丢失；必须使用非阻塞套接字配合循环读取
- **epoll_wait返回后仍遍历所有注册的套接字**: 浪费了epoll仅返回事件套接字的优势，应只处理返回的事件
- **忘记EPOLLET标志后又以非阻塞方式读取**: 混用LT和ET的逻辑会导致数据处理错误

## Code Examples
```c
/* epoll服务器端核心逻辑 */
int epfd, event_cnt;
struct epoll_event *ep_events;
struct epoll_event event;

/* 创建epoll实例 */
epfd = epoll_create(EPOLL_SIZE);
ep_events = malloc(sizeof(struct epoll_event) * EPOLL_SIZE);

/* 将服务器套接字注册到epoll实例 */
event.events = EPOLLIN;       /* 条件触发模式 */
event.data.fd = serv_sock;
epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);

/* 事件循环 */
while (1)
{
    /* 等待事件发生，仅返回有事件的套接字 */
    event_cnt = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);
    if (event_cnt == -1) {
        puts("epoll_wait() error");
        break;
    }

    /* 只遍历发生事件的套接字 */
    for (i = 0; i < event_cnt; i++) {
        if (ep_events[i].data.fd == serv_sock) {
            /* 新连接请求 */
            clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
            event.events = EPOLLIN;
            event.data.fd = clnt_sock;
            epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);
        } else {
            /* 已连接客户端的数据 */
            str_len = read(ep_events[i].data.fd, buf, BUF_SIZE);
            if (str_len == 0) {
                /* 客户端断开连接 */
                epoll_ctl(epfd, EPOLL_CTL_DEL, ep_events[i].data.fd, NULL);
                close(ep_events[i].data.fd);
            } else {
                write(ep_events[i].data.fd, buf, str_len);  /* 回声 */
            }
        }
    }
}
close(epfd);
```
- **What it demonstrates**: epoll服务器的完整工作流程——创建实例、注册套接字、事件循环中仅处理发生事件的套接字

## Reference Tables
| 函数 | 作用 | 返回值 |
|------|------|--------|
| epoll_create(int size) | 创建epoll实例 | 成功epfd，失败-1 |
| epoll_ctl(int epfd, int op, int fd, struct epoll_event *event) | 控制(添加/修改/删除)监视对象 | 成功0，失败-1 |
| epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout) | 等待事件 | 事件数，超时返回0 |

| epoll_ctl操作 | 说明 |
|--------------|------|
| EPOLL_CTL_ADD | 添加新文件描述符到epoll实例 |
| EPOLL_CTL_MOD | 修改已注册文件描述符的事件类型 |
| EPOLL_CTL_DEL | 从epoll实例中删除文件描述符 |

| 特性 | select | epoll |
|------|--------|-------|
| 文件描述符限制 | FD_SETSIZE(通常1024) | 无实质限制 |
| 事件返回方式 | 全量遍历检查 | 仅返回事件套接字 |
| 注册方式 | 每次调用需重新注册 | 一次注册持续有效 |
| 内核支持 | 用户空间轮询 | 内核事件回调 |
| 触发模式 | 仅条件触发 | 支持条件触发和边缘触发 |

| 触发模式 | 通知时机 | 优点 | 缺点 |
|---------|---------|------|------|
| 条件触发(LT) | 缓冲有数据时持续通知 | 不易丢数据 | 可能频繁通知 |
| 边缘触发(ET) | 状态变化时通知一次 | 效率高，通知少 | 需非阻塞读取全部数据 |

## Key Takeaways
1. epoll解决了select的两大缺陷：文件描述符数量限制和O(n)全量遍历
2. epoll的文件描述符注册只需一次，操作系统在内核维护事件表，无需每次调用重复传递
3. 条件触发（LT）是默认模式，简单安全但可能频繁通知；边缘触发（ET）高效但需配合非阻塞I/O
4. 边缘触发模式下必须使用非阻塞套接字，并通过循环读取（errno == EAGAIN）确保读完全部数据
5. epoll_wait仅返回发生事件的套接字列表，处理效率与连接数无关，适合高并发场景

## Connects To
- **Ch 12**: select是I/O多路复用的基础模型，epoll是其高性能进化版
- **Ch 21**: Windows的WSAEventSelect是类似epoll的异步通知I/O模型，两者设计理念相似
- **Ch 23**: IOCP是Windows平台的终极I/O模型，与Linux的epoll形成跨平台对比
