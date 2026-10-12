# Chapter 18: 多线程服务器端的实现

## Core Idea
多线程服务器通过为每个客户端连接创建独立线程实现并发处理；线程相比进程具有更低的创建开销和共享内存空间的优势，但引入了同步问题，需要通过互斥量（mutex）和信号量（semaphore）保护临界区。

## Frameworks Introduced
- **pthread线程框架**: POSIX线程库，提供线程创建、等待、分离等完整功能
  - When to use: Linux平台下需要并发处理多个客户端请求的服务器
  - How: pthread_create创建线程 -> 线程函数处理业务 -> pthread_join/pthread_detach回收

- **互斥量（Mutex）**: 保护临界区的锁机制，确保同一时刻仅一个线程访问共享资源
  - When to use: 多线程访问共享全局变量或共享数据结构时
  - How: pthread_mutex_init初始化 -> pthread_mutex_lock加锁 -> 操作共享数据 -> pthread_mutex_unlock解锁 -> pthread_mutex_destroy销毁

- **信号量（Semaphore）**: 基于计数的同步机制，控制同时访问资源的线程数量
  - When to use: 需要控制资源并发访问数量，或线程间顺序协调时
  - How: sem_init初始化 -> sem_wait获取(计数减1) -> sem_post释放(计数加1)

## Key Concepts
- **线程（Thread）**: 进程内的执行流，共享进程的内存空间
- **pthread_create**: 创建新线程的函数
- **pthread_join**: 等待指定线程终止并回收资源的函数
- **pthread_detach**: 将线程设为分离状态，终止时自动回收资源
- **临界区（Critical Section）**: 多线程同时访问会导致数据不一致的代码区域
- **互斥量（Mutex）**: Mutual Exclusion，确保互斥访问的锁
- **信号量（Semaphore）**: 基于整数计数的同步原语
- **竞态条件（Race Condition）**: 多线程并发访问共享资源导致数据不一致的问题

## Mental Models
- Use pthread_create when you need to handle each client connection independently
- Use pthread_detach when you don't need the thread's return value and want automatic cleanup
- Use mutex when multiple threads modify the same global variable
- Use semaphores when you need to coordinate the order of execution between threads

## Anti-patterns
- **不加锁直接访问全局共享变量**: 导致竞态条件，每次运行结果不同且不可预测
- **获取互斥量后忘记释放**: 导致死锁，其他等待该互斥量的线程永久阻塞
- **将整个循环放入临界区**: 虽然结果正确，但严重降低并发性能，应仅保护共享数据操作

## Code Examples
```c
/* 多线程回声服务器端核心逻辑 */
int clnt_socks[MAX_CLNT];   /* 客户端套接字数组 */
int clnt_cnt = 0;           /* 客户端数量 */
pthread_mutex_t mutx;        /* 互斥量 */

int main(int argc, char *argv[])
{
    pthread_t t_id;
    pthread_mutex_init(&mutx, NULL);  /* 初始化互斥量 */

    /* ... 套接字创建、bind、listen ... */

    while (1)
    {
        clnt_adr_sz = sizeof(clnt_adr);
        clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);

        /* 加锁保护共享的客户端数组 */
        pthread_mutex_lock(&mutx);
        clnt_socks[clnt_cnt++] = clnt_sock;
        pthread_mutex_unlock(&mutx);

        /* 为每个客户端创建线程 */
        pthread_create(&t_id, NULL, handle_clnt, (void*)&clnt_sock);
        pthread_detach(t_id);  /* 分离线程，自动回收 */

        printf("Connected client IP: %s \n", inet_ntoa(clnt_adr.sin_addr));
    }
    close(serv_sock);
    return 0;
}

/* 线程处理函数 - 处理客户端消息 */
void *handle_clnt(void *arg)
{
    int clnt_sock = *((int*)arg);
    int str_len = 0, i;
    char msg[BUF_SIZE];

    while ((str_len = read(clnt_sock, msg, sizeof(msg))) != 0)
        send_msg(msg, str_len);  /* 广播消息给所有客户端 */

    /* 客户端断开时从数组中移除 */
    pthread_mutex_lock(&mutx);
    for (i = 0; i < clnt_cnt; i++) {
        if (clnt_sock == clnt_socks[i]) {
            while (i++ < clnt_cnt - 1)
                clnt_socks[i] = clnt_socks[i + 1];
            break;
        }
    }
    clnt_cnt--;
    pthread_mutex_unlock(&mutx);
    close(clnt_sock);
    return NULL;
}
```
- **What it demonstrates**: 多线程服务器的完整模式——为每个客户端创建线程、用互斥量保护共享的客户端套接字数组、线程终止时安全移除客户端

## Reference Tables
| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| pthread_create(pthread_t *, attr, func, arg) | <pthread.h> | 创建线程 | 成功0，失败错误码 |
| pthread_join(pthread_t, void **) | <pthread.h> | 等待线程终止 | 成功0，失败错误码 |
| pthread_detach(pthread_t) | <pthread.h> | 分离线程 | 成功0，失败错误码 |
| pthread_mutex_init(mutex, attr) | <pthread.h> | 初始化互斥量 | 成功0，失败错误码 |
| pthread_mutex_lock(mutex) | <pthread.h> | 加锁 | 成功0，失败错误码 |
| pthread_mutex_unlock(mutex) | <pthread.h> | 解锁 | 成功0，失败错误码 |
| sem_init(sem, pshared, value) | <semaphore.h> | 初始化信号量 | 成功0，失败-1 |
| sem_wait(sem) | <semaphore.h> | 获取信号量(减1) | 成功0，失败-1 |
| sem_post(sem) | <semaphore.h> | 释放信号量(加1) | 成功0，失败-1 |

| 同步方式 | 适用场景 | 特点 |
|---------|---------|------|
| 互斥量 | 保护临界区，互斥访问 | 二元状态，同一时刻仅一个线程持有 |
| 信号量 | 控制并发访问数量 | 计数式，可允许N个线程同时访问 |
| 二元信号量 | 类似互斥量 | 值为0或1，但可用于进程间同步 |

| 线程回收方式 | 调用者 | 是否阻塞 | 适用场景 |
|-------------|--------|---------|---------|
| pthread_join | 主线程 | 阻塞等待 | 需要获取线程返回值 |
| pthread_detach | 任意线程 | 不阻塞 | 无需返回值，自动回收 |

## Key Takeaways
1. 线程比进程创建开销小（共享内存空间），但需要同步机制保护共享数据
2. pthread_create的第三个参数是线程main函数地址，第四个参数是传递给线程函数的参数
3. pthread_detach分离线程后，线程终止时系统自动回收资源，无需主线程调用join
4. 互斥量用于保护临界区，获取锁后必须确保在所有代码路径上释放锁，避免死锁
5. 信号量通过计数控制资源访问，sem_wait减1阻塞、sem_post加1唤醒，可用于线程间顺序协调
6. 多线程服务器中，共享的客户端连接信息必须用互斥量保护

## Connects To
- **Ch 19**: Windows平台使用CreateThread/_beginthreadex实现线程，概念与Linux的pthread对应
- **Ch 20**: Windows平台使用CRITICAL_SECTION和Mutex/Semaphore实现同步，与Linux的mutex/semaphore对应
- **Ch 24**: HTTP服务器采用多线程模型，为每个客户端请求创建线程处理HTTP协议
