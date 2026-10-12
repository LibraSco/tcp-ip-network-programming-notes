# Chapter 12: I/O 复用(基于 select 的多路复用服务器端)

## Core Idea
I/O 复用(Multiplexing)用 1 个进程同时监视多个文件描述符,通过 `select` 函数集中管理所有套接字的"接收/传输/异常"事件,从而以单进程为任意数量客户端提供服务,避免了多进程模型创建进程的高昂代价。

## Frameworks Introduced
- **select 函数**: 把多个文件描述符集中到一起统一监视的 I/O 复用函数,可同时监视"是否有待读取数据""是否可无阻塞传输""是否发生异常"三类事件;Windows/Linux 同名同功能,移植性好。
  - When to use: 想用 1 个进程服务多个客户端、减少进程创建开销时;是多进程并发服务器的第二种实现方法。
  - How: 设置 fd_set 注册监视对象 → 指定监视范围(最大 fd+1)与超时 → 调用 select 阻塞等待事件 → 查看返回结果用 FD_ISSET 找出发生事件的描述符。
- **fd_set 位操作框架**: fd_set 是存有 0/1 的位数组,第 N 位为 1 表示文件描述符 N 是监视对象;用四个宏完成注册/清除/验证。
  - When to use: 向 select 注册或解析监视对象集合时。
  - How: FD_ZERO 全清零 → FD_SET 注册 → (select 调用后)FD_ISSET 验证哪些发生了事件 → FD_CLR 取消注册。

## Key Concepts
- **I/O 复用(Multiplexing)**: 在 1 个通信频道中传递多个数据(信号)的技术;服务器端引入它可用最少物理要素(1 进程)传递最多数据,减少进程数。
- **select 函数**: `int select(int maxfd, fd_set* readset, fd_set* writeset, fd_set* exceptset, const struct timeval* timeout)`,成功返回发生事件的文件描述符数,超时返回 0,失败返回 -1。
- **maxfd**: 监视对象文件描述符的数量,即注册在 fd_set 中的最大文件描述符值 +1(因描述符从 0 开始)。
- **readset/writeset/exceptset**: 分别注册关注"待读取""可无阻塞传输""发生异常"的文件描述符集合。
- **timeval 结构体**: `{long tv_sec; long tv_usec;}`,设置 select 超时(秒/微秒),防止无限阻塞;传 NULL 表示不超时。
- **事件(Event)**: 监视项对应的情况(如"套接字接收数据")发生时称"发生了事件";select 返回正值即事件数。
- **fd_set**: 存有 0 和 1 的位数组,用于集中管理待监视的文件描述符集合。
- **FD_ZERO**: 将 fd_set 变量所有位初始化为 0。
- **FD_SET**: 在 fd_set 中注册某文件描述符(对应位设 1)。
- **FD_CLR**: 从 fd_set 中清除某文件描述符(对应位设 0)。
- **FD_ISSET**: 若 fd_set 中包含某文件描述符(位为 1)则返回真,用于验证 select 调用结果。

## Mental Models
- Use **select** when 需要用 1 个进程同时服务多个客户端,且能接受"事件到达才处理"的事件驱动模型。
- Use **fd_set + 四宏** when 需要把监视对象集中注册/解析:FD_ZERO 初始化、FD_SET 注册、FD_ISSET 查结果、FD_CLR 注销。
- Use **复制 fd_set 到临时变量** when 每次 select 前都要备份原集合,因为 select 会把未发生事件的位清零、破坏原注册信息——这是 select 的通用方法。
- Use **每次循环内重新初始化 timeout** when 设置超时,因为 select 调用后 timeval 成员会被替换为超时前剩余时间,必须在每次调用前重置。

## Anti-patterns
- **把原 fd_set 直接传给 select 不备份**: select 会清除未发生事件描述符的位,导致原注册信息丢失,之后无法继续监视——必须复制到临时变量。
- **在循环外只设一次 timeout**: timeout 在 select 后被改为剩余时间,不重置会导致后续超时不准。
- **过度依赖 I/O 复用模型**: 该方案并非适用于所有情况,应根据服务器端特点选用不同实现;且 select 有 fd 数量上限与每次全量拷贝的开销。
- **不在注册新客户端后更新 fd_max**: 新 accept 的 clnt_sock 可能大于当前 fd_max,需 `if(fd_max<clnt_sock) fd_max=clnt_sock;` 否则监视范围不足。

## Code Examples
```c
/* echo_selectserv.c —— 基于 select 的 I/O 复用回声服务器(核心片段) */
int main(int argc, char *argv[])
{
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    struct timeval timeout;
    fd_set reads, cpy_reads;          /* reads:原集合;cpy_reads:每次备份 */
    socklen_t adr_sz;
    int fd_max, str_len, fd_num, i;
    char buf[BUF_SIZE];

    serv_sock=socket(PF_INET, SOCK_STREAM, 0);
    /* ... bind / listen ... */

    FD_ZERO(&reads);
    FD_SET(serv_sock, &reads);        /* 注册服务器套接字(监听连接请求) */
    fd_max=serv_sock;

    while(1)
    {
        cpy_reads=reads;              /* 关键:备份,因 select 会破坏原集合 */
        timeout.tv_sec=5;
        timeout.tv_usec=0;            /* 关键:每次循环重新初始化超时 */

        if((fd_num=select(fd_max+1, &cpy_reads, 0, 0, &timeout))==-1)
            break;
        if(fd_num==0)
            continue;                /* 超时 */

        for(i=0; i<fd_max+1; i++)
        {
            if(FD_ISSET(i, &cpy_reads))
            {
                if(i==serv_sock)     /* 服务器套接字有变化=新连接请求 */
                {
                    adr_sz=sizeof(clnt_adr);
                    clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
                    FD_SET(clnt_sock, &reads);     /* 注册新客户端 */
                    if(fd_max<clnt_sock)
                        fd_max=clnt_sock;           /* 更新监视范围 */
                    printf("connected client: %d \n", clnt_sock);
                }
                else                 /* 客户端套接字有变化=有数据/EOF */
                {
                    str_len=read(i, buf, BUF_SIZE);
                    if(str_len==0)   /* EOF=关闭请求 */
                    {
                        FD_CLR(i, &reads);          /* 从集合移除 */
                        close(i);
                        printf("closed client: %d \n", i);
                    }
                    else
                        write(i, buf, str_len);     /* 回声服务 */
                    }
                }
            }
        }
    }
    close(serv_sock);
    return 0;
}
```
- **What it demonstrates**: select 用单进程监视服务器套接字+所有客户端套接字;服务器套接字事件即新连接,客户端套接字事件即数据或 EOF;备份集合与重置超时是通用要点。

## Reference Tables

| select 参数 | 含义 |
|-----------|------|
| `maxfd` | 监视对象文件描述符数量(最大 fd +1) |
| `readset` | 关注"是否有待读取数据"的描述符集合 |
| `writeset` | 关注"是否可无阻塞传输"的描述符集合 |
| `exceptset` | 关注"是否发生异常"的描述符集合 |
| `timeout` | 超时信息,防止无限阻塞;NULL 表示不超时 |

| select 返回值 | 含义 |
|--------------|------|
| 正整数 | 发生事件的文件描述符数 |
| 0 | 超时 |
| -1 | 发生错误 |

| fd_set 宏 | 功能 |
|-----------|------|
| `FD_ZERO(fd_set* fdset)` | 所有位初始化为 0 |
| `FD_SET(int fd, fd_set* fdset)` | 注册文件描述符 fd(位置 1) |
| `FD_CLR(int fd, fd_set* fdset)` | 清除文件描述符 fd(位置 0) |
| `FD_ISSET(int fd, fd_set* fdset)` | 若包含 fd 返回真(验证结果) |

| 并发模型 | 进程数 | 代价 | IPC 需求 |
|---------|--------|------|----------|
| 多进程服务器(Ch10) | 每客户端 1 个 | 高(创建进程+独立内存) | 需要(管道) |
| I/O 复用服务器(Ch12) | 仅 1 个 | 低 | 不需要(共享内存) |

## Key Takeaways
1. I/O 复用是并发服务器的第二种方法:用 1 个进程集中监视所有套接字,减少进程数,避免多进程的高代价与 IPC 复杂度。
2. select 调用三步走:① 设置 fd_set 注册监视对象并指定 maxfd+1 与 timeout;② 调用 select 阻塞等待事件;③ 用 FD_ISSET 遍历找出发生事件的描述符处理。
3. 每次调用 select 前必须把原 fd_set 复制到临时变量,因为 select 会清除未发生事件的位;同时每次都要重新初始化 timeout(它会被改为剩余时间)。
4. 服务器套接字也是 select 的监视对象:它"有接收数据"等价于"有新连接请求",从而 accept;新客户端套接字需 FD_SET 注册并更新 fd_max。
5. fd_set 是 0/1 位数组,第 N 位为 1 表示描述符 N 是监视对象;四位宏以位为单位操作,无需手动设位。
6. select 在 Windows/Linux 同名同功能,移植性好;但并非适用于所有场景,应据服务器特点选择模型,且 select 存在 fd 上限与全量拷贝开销。

## Connects To
- **Ch 10**: I/O 复用是与多进程服务器并列的并发实现法;select 用 1 进程服务多客户端,正好弥补多进程"创建进程代价高、需 IPC"的缺点。
- **Ch 13**: select 的 exceptset 用于监视"异常",与第 13 章 MSG_OOB 紧急数据(SIGURG)的接收机制相关。
- **Ch 17**: 本章提到的"事件"概念在第 17 章异步通知 I/O 模型中正式使用;select 是更底层的事件驱动基础。
- **POSIX select / fd_set**: select 与 fd_set 是跨平台标准 I/O 复用接口(BSD 起源),Windows 下也提供同名实现。
