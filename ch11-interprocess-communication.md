# Chapter 11: 进程间通信(Inter Process Communication, IPC)

## Core Idea
进程拥有完全独立的内存空间,无法直接共享数据;进程间通信(尤其是管道 pipe)通过操作系统提供的、两个进程可同时访问的内存空间来交换数据,而实现双向通信时必须创建 2 个管道。

## Frameworks Introduced
- **管道(pipe)**: 一种 IPC 技术,通过 `pipe(int filedes[2])` 创建一对文件描述符,`filedes[0]` 为出口(读)、`filedes[1]` 为入口(写),供父子进程进行数据交换。
  - When to use: 具有父子关系的进程需要交换数据时;fork 后父子进程共享同一管道的文件描述符。
  - How: 父进程 pipe 创建管道 → fork(子进程复制文件描述符) → 各自关闭不需要的一端 → 一方 write(fds[1])、另一方 read(fds[0])。
- **双向通信(2 个管道)模型**: 用 1 个管道做双向通信会因"先读的进程取走数据"而出错,因此需创建 2 个管道,各负责一个方向的数据流动。
  - When to use: 父子进程需要相互收发数据(双向),且不能预测/控制运行流程时。
  - How: pipe(fds1) + pipe(fds2) → fork → 子进程用 fds1 向父发、用 fds2 从父收;父进程反之。
- **管道分割服务器 I/O**: 服务器用一个子进程专门从管道读取并保存所有客户端的回声消息(如写入文件),其他子进程处理客户端连接并通过管道写入数据。
  - When to use: 需要把所有客户端交互数据汇总保存(如日志/存储)的多进程服务器。
  - How: fork 一个"保存子进程"读管道写文件 → 主循环 fork 多个"服务子进程"各处理一个客户端,并向管道写入数据。

## Key Concepts
- **进程间通信(IPC)**: 两个不同进程间交换数据的能力,前提是操作系统提供两进程可同时访问的内存空间。
- **pipe 函数**: `int pipe(int filedes[2])`,创建管道并返回两个文件描述符;成功返回 0,失败返回 -1。
- **filedes[0]**: 管道出口,通过它接收(读)数据。
- **filedes[1]**: 管道入口,通过它传输(写)数据。
- **单向通信**: 1 个管道中数据单向流动(一端只写、另一端只读)。
- **双向通信**: 父子进程相互收发数据;1 个管道做双向会出错,需 2 个管道。
- **无主数据**: 数据进入管道后与发送者脱钩,先调用 read 的进程(无论是不是发送者)将取走数据——这是单管道双向通信失败的根因。
- **文件描述符复制(管道场景)**: fork 复制的是管道的文件描述符而非管道本身,因此 fork 后父子进程同时拥有同一管道的进出口。

## Mental Models
- Use **pipe + fork** when 需要在父子进程间单向传递数据(如子→父),先 pipe 再 fork 使子进程复制得到同一对描述符。
- Use **2 个管道** when 父子进程需双向通信——1 个管道无法避免"先读进程取走自己刚发的数据",2 个管道让每个方向独立,无需预测/控制运行流程。
- Use **管道分割 I/O** when 多进程服务器需要把所有客户端数据汇总到一处保存:开一个专门读管道、写文件的子进程,其余服务子进程向管道写数据。
- Use **关闭不需要的描述符端** when 单向通信中,发送方关闭读端、接收方关闭写端,以保证数据流方向清晰、避免误读。

## Anti-patterns
- **用 1 个管道做双向通信**: 数据进入管道后变为无主数据,先 read 的进程会取走自己刚 write 的数据,导致另一方无限阻塞等待——必须改用 2 个管道。
- **注释掉同步用的 sleep 后期望单向管道仍正常**: pipe2.c 中去掉 sleep(2) 会暴露单管道双向通信的竞态问题,说明单管道方案依赖时序、不可靠。
- **认为 fork 复制了管道本身**: 实际复制的只是管道的 I/O 文件描述符,父子进程共享同一管道,关闭描述符不影响另一进程的描述符。

## Code Examples
```c
/* pipe3.c —— 用 2 个管道实现父子进程双向通信 */
#include <stdio.h>
#include <unistd.h>
#define BUF_SIZE 30

int main(int argc, char *argv[])
{
    int fds1[2], fds2[2];
    char str1[]="Who are you?";
    char str2[]="Thank you for your message";
    char buf[BUF_SIZE];
    pid_t pid;

    pipe(fds1), pipe(fds2);          /* 创建 2 个管道 */
    pid=fork();
    if (pid==0)
    {
        write(fds1[1], str1, sizeof(str1));   /* 子→父(经管道1) */
        read(fds2[0], buf, BUF_SIZE);        /* 子←父(经管道2) */
        printf("Child proc output: %s \n", buf);
    }
    else
    {
        read(fds1[0], buf, BUF_SIZE);         /* 父←子(经管道1) */
        printf("Parent proc output: %s \n", buf);
        write(fds2[1], str2, sizeof(str2));   /* 父→子(经管道2) */
        sleep(3);
    }
    return 0;
}
```
```c
/* echo_storeserv.c 片段 —— 用管道把回声数据汇总保存到文件 */
/* 父进程:pipe(fds) 后 fork 一个"保存子进程" */
pipe(fds);
pid=fork();
if (pid==0)                       /* 保存子进程:循环读管道并写文件 */
{
    FILE * fp=fopen("echomsg.txt", "wt");
    char msgbuf[BUF_SIZE];
    int i, len;
    for(i=0; i<10; i++)
    {
        len=read(fds[0], msgbuf, BUF_SIZE);
        fwrite((void*)msgbuf, 1, len, fp);
    }
    fclose(fp);
    return 0;
}
/* 之后 accept 客户端时再 fork 服务子进程,既回声又向管道写数据 */
while((str_len=read(clnt_sock, buf, BUF_SIZE))!=0)
{
    write(clnt_sock, buf, str_len);   /* 回声给客户端 */
    write(fds[1], buf, str_len);      /* 同时写入管道供保存 */
}
```
- **What it demonstrates**: 管道 1 出 1 入的原理;2 个管道实现可靠双向通信;以及把管道用于服务器数据汇总存储。

## Reference Tables

| 管道描述符 | 角色 | 操作 |
|-----------|------|------|
| `filedes[0]` | 管道出口 | read 接收数据 |
| `filedes[1]` | 管道入口 | write 传输数据 |

| 通信需求 | 管道数量 | 原因 |
|----------|---------|------|
| 单向(子→父 或 父→子) | 1 个 | 一端只写、一端只读即可 |
| 双向(父子互相收发) | 2 个 | 单管道数据无主,先读者取走导致死等 |

| pipe 返回值 | 含义 |
|------------|------|
| 0 | 成功创建管道 |
| -1 | 创建失败 |

## Key Takeaways
1. 进程内存完全独立(连 fork 出的子进程也不共享内存),IPC 必须借助操作系统提供的共享内存空间(管道)完成。
2. pipe 创建管道返回一对描述符:[0]读/[1]写;fork 后父子进程共享同一管道的描述符,这是管道 IPC 的基础。
3. 双向通信必须用 2 个管道:1 个管道中数据"无主",先 read 的进程会取走自己刚发的数据,导致对方无限阻塞。
4. pipe3.c 用 fds1(子→父)+ fds2(父→子)两个管道,使每个方向独立流动,无需预测或控制运行流程,是双向通信的正确做法。
5. 管道可用来分割服务器 I/O:开一个专门读管道并写文件的子进程,多个服务子进程把回声数据写入管道,实现数据汇总存储。
6. IPC 虽与构建服务器无直接关系,但有助于理解操作系统,并能构建多种类型服务器。

## Connects To
- **Ch 10**: 第 10 章 fork 创建的子进程内存独立、不与父进程共享,这正是本章 IPC 产生的原因;echo_storeserv.c 复用了第 10 章的 echo_mpserv.c 框架。
- **Ch 12**: 多进程服务器需 IPC 交换数据、创建进程代价高,I/O 复用(select)用 1 个进程减少进程数、规避 IPC。
- **POSIX pipe**: 管道是 UNIX/Linux 标准 IPC 机制,基于文件描述符,遵循 read/write 语义。
