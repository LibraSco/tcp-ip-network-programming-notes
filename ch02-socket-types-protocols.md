# Chapter 2: 套接字类型与协议设置

## Core Idea
socket函数的三个参数(协议族domain、数据传输方式type、具体协议protocol)共同决定套接字的特性;同一种协议族中存在多种数据传输方式,因此需要type参数进一步指定。

## Frameworks Introduced
- **协议族选择框架(Protocol Family)**: 通过socket函数第一个参数domain指定套接字使用的协议族,它决定通信范围(IPv4/IPv6/本地等)。
  - When to use: 创建套接字时确定网络协议范围
  - How: 传递PF_INET(IPv4)、PF_INET6(IPv6)、PF_LOCAL(本地UNIX)等常量

- **数据传输方式选择框架(Type)**: 通过socket函数第二个参数type决定套接字的数据传输特性(面向连接或面向消息)。
  - When to use: 在确定协议族后选择数据传输方式
  - How: 传递SOCK_STREAM(面向连接)或SOCK_DGRAM(面向消息)

- **协议确定框架(Protocol)**: 通过socket函数第三个参数protocol在指定协议族和传输方式范围内选择最终通信协议,通常传0表示使用默认协议。
  - When to use: 当domain+type能唯一确定协议时传0,需要指定时传递协议编号
  - How: PF_INET + SOCK_STREAM + 0 = TCP; PF_INET + SOCK_DGRAM + 0 = UDP

## Key Concepts
- **协议(Protocol)**: 计算机间对话必备的通信规则,为了完成数据交换而定好的约定
- **协议族(Protocol Family)**: 套接字通信中使用的协议分类信息,如PF_INET对应IPv4互联网协议族
- **SOCK_STREAM**: 面向连接的套接字,提供可靠的、按序传递的、基于字节的传输方式
- **SOCK_DGRAM**: 面向消息的套接字,提供不可靠的、高速的、基于消息的传输方式
- **数据边界(Boundary)**: 面向连接的套接字不存在数据边界(可多次write一次read),面向消息的套接字存在数据边界(每次发送的数据独立保留)
- **套接字缓冲(Buffer)**: 收发数据的套接字内部由字节数组构成的缓冲,用于暂存数据
- **流控制**: 面向连接的套接字根据接收端状态传输数据,缓冲满时传输端停止,不会丢失数据

## Mental Models
- Use "conveyor belt with two workers" when understanding SOCK_STREAM: one belt guarantees ordered, lossless delivery; data accumulates in buffer and can be read in any number of calls (no boundary)
- Use "courier delivering parcels" when understanding SOCK_DGRAM: each message is an independent parcel, delivery is not guaranteed, order may change, but each parcel preserves its boundary
- Use "domain narrows the scope, type picks the transport style, protocol picks the exact protocol" when reasoning about why socket() needs three parameters

## Anti-patterns
- **认为协议族决定了一切**: 确定协议族(PF_INET)并不能同时决定数据传输方式,同一协议族中存在多种传输方式,还需type参数
- **面向连接套接字中按write次数读取**: TCP不存在数据边界,3次write的100字节可能被1次read全部读取,不能假设read次数等于write次数

## Code Examples
```c
// fd_seri.c —— 验证文件描述符与套接字的关系
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/socket.h>

int main(void)
{
    int fd1, fd2, fd3;
    fd1 = socket(PF_INET, SOCK_STREAM, 0);      // 创建TCP套接字
    fd2 = open("test.dat", O_CREAT|O_WRONLY|O_TRUNC);  // 创建文件
    fd3 = socket(PF_INET, SOCK_DGRAM, 0);        // 创建UDP套接字

    printf("file descriptor 1: %d\n", fd1);      // 输出 3
    printf("file descriptor 2: %d\n", fd2);      // 输出 4
    printf("file descriptor 3: %d\n", fd3);      // 输出 5

    close(fd1); close(fd2); close(fd3);
    return 0;
}
```
- **What it demonstrates**: 文件和套接字都分配文件描述符,且按创建顺序从3开始连续编号(SOCK_STREAM和SOCK_DGRAM的区别)

## Reference Tables

| 协议族名称 | 协议族 |
|-----------|--------|
| PF_INET | IPv4互联网协议族 |
| PF_INET6 | IPv6互联网协议族 |
| PF_LOCAL | 本地通信的UNIX协议族 |
| PF_PACKET | 底层套接字的协议族 |
| PF_IPX | IPX Novell协议族 |

| 特性 | SOCK_STREAM(面向连接) | SOCK_DGRAM(面向消息) |
|------|----------------------|---------------------|
| 数据丢失 | 传输中不丢失 | 可能丢失 |
| 传输顺序 | 按序传输 | 不保证顺序 |
| 数据边界 | 不存在(无边界) | 存在(有边界) |
| 连接关系 | 必须一一对应 | 无需连接 |
| 可靠性 | 可靠(基于TCP) | 不可靠(基于UDP) |
| 典型协议 | TCP | UDP |

| socket参数组合 | 结果 |
|---------------|------|
| PF_INET + SOCK_STREAM + 0 | TCP套接字 |
| PF_INET + SOCK_DGRAM + 0 | UDP套接字 |

## Key Takeaways
1. socket函数三参数是层递关系:domain选协议族范围,type选传输方式,protocol选最终协议(传0用默认)
2. SOCK_STREAM(TCP)三大特征:数据不丢失、按序传输、不存在数据边界——因为内部有缓冲且提供流控制
3. SOCK_DGRAM(UDP)三大特征:可能丢失数据、不保证顺序、存在数据边界——每次传输的数据独立成包
4. 面向连接的套接字必须一一对应,一个服务器端套接字只能连接一个客户端套接字
5. 缓冲满时不会丢数据,因为传输端会暂停发送直到接收端read释放空间(流控制)

## Connects To
- **Ch 1**: socket函数在第1章仅简单介绍,本章深入解释其三个参数
- **Ch 4**: SOCK_STREAM(TCP)的完整服务器/客户端实现详见第4章
- **Ch 6**: SOCK_DGRAM(UDP)的服务器/客户端实现详见第6章
- **OSI 7层模型**: 本章的4层协议栈简化了OSI 7层模型,对程序员4层足够
