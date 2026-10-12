# Chapter 10: 多进程服务器端的实现

## Core Idea
通过 `fork` 函数创建子进程,使服务器端能为每个连接的客户端分配独立进程同时提供服务;但必须处理好僵尸进程的回收(通过 `wait`/`waitpid` 与 `SIGCHLD` 信号)以及套接字文件描述符的复制与关闭。

## Frameworks Introduced
- **并发服务器(多进程模型)**: 每当有客户端连接请求,服务器端(父进程)调用 `fork` 创建子进程,由子进程负责与该客户端通信,父进程继续受理新的连接。
  - When to use: 需要同时向多个客户端提供服务、且运行在 Linux 平台(Windows 不支持 `fork`)。
  - How: accept 获取客户端套接字 → fork 创建子进程 → 子进程关闭服务器套接字、父进程关闭客户端套接字 → 子进程提供回声服务 → 父进程通过 SIGCHLD 信号回收子进程。
- **进程回收框架(wait/waitpid + 信号)**: 子进程终止时操作系统产生 `SIGCHLD` 信号,父进程在信号处理函数中调用 `waitpid` 获取子进程退出状态并销毁僵尸进程。
  - When to use: 任何创建子进程的程序,为防止僵尸进程占用系统资源。
  - How: 注册 `SIGCHLD` 信号处理函数 → 在处理函数中调用 `waitpid(-1, &status, WNOHANG)` → 用 `WIFEXITED`/`WEXITSTATUS` 宏解析状态。

## Key Concepts
- **进程(Process)**: 占用内存空间、正在运行的程序;操作系统进行程序运行的基本单位。
- **进程ID(PID)**: 操作系统分配给每个进程的大于 1 的整数标识符,可用 `ps au` 查看。
- **fork 函数**: 创建调用进程的副本(子进程);父进程中返回子进程 PID,子进程中返回 0,失败返回 -1。
- **僵尸进程(Zombie)**: 子进程已终止(执行完 main 或调用 exit),但其退出状态值尚未被父进程取走,操作系统仍保留其信息的进程状态,会占用系统资源。
- **wait 函数**: 阻塞式获取已终止子进程的状态,成功返回终止子进程 PID;若没有已终止子进程则阻塞等待。
- **waitpid 函数**: 可指定 PID、非阻塞(`WNOHANG`)地获取子进程状态,防止程序阻塞。
- **WIFEXITED 宏**: 判断子进程是否正常终止,返回真。
- **WEXITSTATUS 宏**: 返回子进程正常终止时的返回值(exit 参数或 main return 值)。
- **signal(信号)**: 操作系统在特定事件发生时向进程传递的消息;`SIGCHLD` 表示子进程终止,`SIGALRM` 表示 alarm 预约时间到,`SIGINT` 表示 Ctrl+C。
- **sigaction 函数**: 比 signal 更可移植、更可靠的信号注册函数,通过 `struct sigaction` 注册信号处理函数。
- **文件描述符复制**: fork 复制的是文件描述符(指向套接字的"把手"),而非套接字本身;一个套接字可有多个文件描述符,全部关闭后套接字才销毁。

## Mental Models
- Use **fork + 子进程** when 需要让多个执行流并行处理多个客户端连接,且能接受进程创建的内存与调度开销。
- Use **SIGCHLD 信号 + waitpid(WNOHANG)** when 需要非阻塞地、自动回收子进程,避免僵尸进程长期占用资源。
- Use **关闭无关套接字文件描述符** when fork 之后,子进程关闭服务器套接字、父进程关闭客户端套接字,以整理成"每个进程只持有自己需要的描述符"状态。
- Use **shutdown(sock, SHUT_WR)** when 在 I/O 分割模型中,write_routine 需向服务器传递 EOF 而不能仅用一次 close(因为 fork 已复制文件描述符)。

## Anti-patterns
- **fork 后不关闭无关文件描述符**: 一个套接字存在 2 个文件描述符,只要有一个未关闭,套接字就无法销毁,连接无法真正断开。
- **用 wait 阻塞式回收子进程**: 在服务器循环中直接调用 wait 会阻塞父进程受理新连接,应改用信号驱动 + waitpid(WNOHANG)。
- **不处理 SIGCHLD 导致僵尸进程堆积**: 子进程终止后若父进程不主动请求退出状态,僵尸进程会持续占用系统内存与 PID 资源。
- **在 Windows 平台使用多进程模型**: Windows 不支持 fork,多进程服务器仅适用于 Linux;Windows 应改用第 18/19 章的线程方案。

## Code Examples
```c
/* echo_mpserv.c —— 基于多进程的并发回声服务器(核心片段) */
int main(int argc, char *argv[])
{
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    pid_t pid;
    struct sigaction act;
    socklen_t adr_sz;
    int str_len, state;
    char buf[BUF_SIZE];

    /* 注册 SIGCHLD 信号处理函数,防止产生僵尸进程 */
    act.sa_handler=read_childproc;
    sigemptyset(&act.sa_mask);
    act.sa_flags=0;
    state=sigaction(SIGCHLD, &act, 0);

    serv_sock=socket(PF_INET, SOCK_STREAM, 0);
    /* ... bind / listen ... */

    while(1)
    {
        adr_sz=sizeof(clnt_adr);
        clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
        if (clnt_sock==-1)
            continue;
        else
            puts("new client connected...");
        pid=fork();
        if (pid==-1)
        {
            close(clnt_sock);
            continue;
        }
        if(pid==0) /* 子进程运行区域 */
        {
            close(serv_sock);              /* 关闭复制来的无关服务器套接字 */
            while((str_len=read(clnt_sock, buf, BUF_SIZE))!=0)
                write(clnt_sock, buf, str_len);
            close(clnt_sock);
            puts("client disconnected...");
            return 0;
        }
        else
            close(clnt_sock);              /* 父进程关闭客户端套接字 */
    }
    close(serv_sock);
    return 0;
}

/* SIGCHLD 信号处理函数:非阻塞回收子进程 */
void read_childproc(int sig)
{
    pid_t pid;
    int status;
    pid=waitpid(-1, &status, WNOHANG);
    printf("removed proc id: %d \n", pid);
}
```
- **What it demonstrates**: accept 后立即 fork,子进程关闭服务器套接字并负责回声服务,父进程关闭客户端套接字并继续受理连接;通过 SIGCHLD 信号异步回收僵尸进程。

## Reference Tables

| 方法 | 功能 | 阻塞? | 适用场景 |
|------|------|------|----------|
| `wait(&status)` | 获取已终止子进程状态 | 阻塞(无子进程终止时等待) | 简单场景 |
| `waitpid(-1, &status, WNOHANG)` | 获取指定子进程状态 | 非阻塞(WNOHANG) | 信号处理函数中 |
| `signal(signo, handler)` | 注册信号处理函数 | — | 旧式,可移植性差 |
| `sigaction(signo, &act, 0)` | 注册信号处理函数 | — | 推荐方式,可移植可靠 |

| 宏 | 作用 |
|----|------|
| `WIFEXITED(status)` | 子进程正常终止返回真 |
| `WEXITSTATUS(status)` | 返回子进程的返回值 |

| fork 返回值 | 含义 |
|------------|------|
| 正整数(父进程) | 子进程 PID |
| 0(子进程) | 当前为子进程 |
| -1 | 创建失败 |

## Key Takeaways
1. 并发服务器的核心是:accept 后 fork,子进程服务客户端,父进程继续 accept,通过 SIGCHLD 信号回收僵尸进程。
2. fork 复制的是文件描述符而非套接字;fork 后必须关闭各进程不需要的描述符,否则套接字无法销毁、连接无法断开。
3. 僵尸进程的根因是父进程未主动请求子进程退出状态;用 sigaction 注册 SIGCHLD,在处理函数中 waitpid(WNOHANG) 回收即可解决。
4. 客户端可通过 fork 分割 I/O:子进程负责发送(write_routine)、父进程负责接收(read_routine),使数据收发解耦、可连续发送。
5. I/O 分割后用 shutdown(sock, SHUT_WR) 而非 close 传递 EOF,因为 fork 已复制文件描述符,单次 close 无法释放套接字。
6. 多进程模型仅适用于 Linux(Windows 不支持 fork);它代价较高(进程创建需大量运算和独立内存空间),后续 I/O 复用与多线程是替代方案。

## Connects To
- **Ch 11**: 多进程之间内存完全独立,需用管道(pipe)等 IPC 方法交换数据,这是第 11 章主题。
- **Ch 12**: 多进程服务器每个连接都创建进程代价高,I/O 复用(select)用 1 个进程服务多客户端,是并发的第二种方法。
- **Ch 16**: I/O 流分离的其他内容,与本章 echo_mpclient 的 shutdown/EOF 传递直接相关。
- **POSIX fork/wait/sigaction**: 这些是 Linux/UNIX 进程控制的标准系统调用。
