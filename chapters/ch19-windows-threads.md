# Chapter 19: Windows平台下线程的使用

## Core Idea
Windows平台通过内核对象（Kernel Object）管理线程资源，CreateThread创建线程并返回句柄（Handle）；内核对象具有signaled/non-signaled两种状态，通过WaitForSingleObject可等待线程终止，实现与Linux pthread类似的线程管理。

## Frameworks Introduced
- **内核对象（Kernel Object）**: Windows操作系统管理的资源信息数据块，包含状态、安全等信息
  - When to use: 创建线程、进程、互斥量等资源时，系统自动创建内核对象
  - How: 通过CreateThread等函数创建资源，获得句柄引用内核对象

- **CreateThread/_beginthreadex线程创建**: Windows平台创建线程的两种方式
  - When to use: CreateThread用于纯Win32编程；_beginthreadex用于需要调用C/C++标准函数的场景
  - How: 传递线程函数地址和参数，返回线程句柄

- **WaitForSingleObject状态等待**: 通过内核对象状态判断线程是否终止的机制
  - When to use: 需要等待线程完成、或设置超时防止永久阻塞时
  - How: 传递句柄和超时时间，线程终止时内核对象变为signaled状态

## Key Concepts
- **内核对象（Kernel Object）**: 操作系统保存资源信息的数据结构，由系统拥有和管理
- **句柄（Handle）**: 用于区分和引用内核对象的整数值，类似Linux的文件描述符
- **signaled状态**: 内核对象的一种状态，表示"事件已发生"（如线程已终止）
- **non-signaled状态**: 内核对象的初始状态，表示"事件未发生"（如线程运行中）
- **auto-reset模式**: 进入signaled后自动回到non-signaled的内核对象
- **manual-reset模式**: 进入signaled后保持不变，需手动重置
- **CreateThread**: Windows原生线程创建函数
- **_beginthreadex**: C运行库提供的线程创建函数，保证C/C++标准函数线程安全
- **WaitForSingleObject**: 等待单个内核对象变为signaled的函数
- **WaitForMultipleObjects**: 等待多个内核对象的函数

## Mental Models
- Use _beginthreadex instead of CreateThread when the thread calls C/C++ standard library functions
- Use WaitForSingleObject with INFINITE when you need to wait for a thread to complete (equivalent to pthread_join)
- Use WaitForMultipleObjects when you need to wait for multiple threads to all complete
- Use signaled/non-signaled states to understand when kernel objects trigger wait functions to return

## Anti-patterns
- **使用CreateThread创建调用C/C++标准函数的线程**: C标准库的strtok、errno等函数在CreateThread创建的线程中使用不稳定，应使用_beginthreadex
- **在main函数返回后不等待线程**: Windows同Linux一样，进程终止会销毁所有线程，需要WaitForSingleObject等待

## Code Examples
```c
/* Windows多线程示例 - 使用_beginthreadex创建线程并等待 */
#include <stdio.h>
#include <windows.h>
#include <process.h>   /* _beginthreadex, _endthreadex */

unsigned WINAPI ThreadFunc(void *arg);

int main(int argc, char *argv[])
{
    HANDLE hThread;
    DWORD wr;
    unsigned threadID;
    int param = 5;

    /* 创建线程 - 使用_beginthreadex而非CreateThread */
    hThread = (HANDLE)_beginthreadex(NULL, 0, ThreadFunc, (void*)&param, 0, &threadID);
    if (hThread == 0) {
        puts("_beginthreadex() error");
        return -1;
    }

    /* 等待线程终止 - 相当于Linux的pthread_join */
    if ((wr = WaitForSingleObject(hThread, INFINITE)) == WAIT_FAILED) {
        puts("thread wait error");
        return -1;
    }

    /* 检查返回原因 */
    printf("wait result: %s \n", (wr == WAIT_OBJECT_0) ? "signaled" : "time-out");
    puts("end of main");
    return 0;
}

unsigned WINAPI ThreadFunc(void *arg)
{
    int i;
    int cnt = *((int*)arg);
    for (i = 0; i < cnt; i++) {
        Sleep(1000);        /* Sleep以1/1000秒为单位 */
        puts("running thread");
    }
    return 0;
}
```
- **What it demonstrates**: 使用_beginthreadex创建线程、WaitForSingleObject等待线程终止的完整模式，体现了Windows线程管理的内核对象状态机制

## Reference Tables
| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| CreateThread(attr, stack, func, param, flags, tid) | <windows.h> | 创建线程 | 成功句柄，失败NULL |
| _beginthreadex(sec, stack, start, arg, flag, tid) | <process.h> | 创建线程(安全) | 成功句柄，失败0 |
| WaitForSingleObject(hHandle, dwMilliseconds) | <windows.h> | 等待内核对象 | WAIT_OBJECT_0/WAIT_TIMEOUT/WAIT_FAILED |
| WaitForMultipleObjects(n, handles, waitAll, timeout) | <windows.h> | 等待多对象 | 对象索引/超时/失败 |
| Sleep(dwMilliseconds) | <windows.h> | 线程休眠 | 无 |

| 内核对象状态 | 含义 | 线程对象示例 |
|-------------|------|------------|
| signaled | 事件已发生 | 线程已终止 |
| non-signaled | 事件未发生 | 线程运行中 |

| WaitForSingleObject返回值 | 含义 |
|--------------------------|------|
| WAIT_OBJECT_0 | 对象变为signaled状态 |
| WAIT_TIMEOUT | 超时返回 |
| WAIT_FAILED | 函数调用失败 |

| 概念 | Linux | Windows |
|------|-------|--------|
| 线程创建 | pthread_create | _beginthreadex |
| 线程等待 | pthread_join | WaitForSingleObject |
| 线程休眠 | sleep(秒) | Sleep(毫秒) |
| 线程ID | pthread_t | unsigned/DWORD |
| 线程句柄 | 无（通过pthread_t） | HANDLE |
| 资源管理 | 文件描述符 | 内核对象+句柄 |

## Key Takeaways
1. Windows通过内核对象管理线程，句柄是引用内核对象的整数值，类似Linux文件描述符
2. 线程内核对象初始为non-signaled状态，线程终止时变为signaled状态
3. _beginthreadex比CreateThread更安全，确保C/C++标准函数在线程中正确工作
4. WaitForSingleObject(hThread, INFINITE)等价于Linux的pthread_join，等待线程终止
5. WaitForMultipleObjects可同时等待多个线程，bWaitAll为TRUE时需全部完成才返回
6. 线程ID跨进程唯一但句柄不跨进程，通常使用句柄而非线程ID管理线程

## Connects To
- **Ch 18**: Linux的pthread线程模型与Windows线程模型概念对应，函数名不同但原理相似
- **Ch 20**: 本章的内核对象状态理论是下一章线程同步（Mutex/Semaphore/Event）的基础
- **Ch 24**: Windows版HTTP服务器使用_beginthreadex为每个客户端创建请求处理线程
