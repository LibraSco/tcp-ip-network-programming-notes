# Chapter 20: Windows平台下的线程同步

## Core Idea
Windows提供用户模式（CRITICAL_SECTION）和内核模式（Mutex/Semaphore/Event）两种同步机制；用户模式同步速度快但功能有限，内核模式同步功能丰富且可跨进程、可设置超时，但需付出模式切换的性能代价。

## Frameworks Introduced
- **CRITICAL_SECTION（用户模式同步）**: 基于用户模式的临界区保护机制，无需操作系统介入
  - When to use: 单进程内多线程同步，追求最大速度且无需超时功能时
  - How: InitializeCriticalSection初始化 -> EnterCriticalSection进入 -> LeaveCriticalSection离开 -> DeleteCriticalSection销毁

- **Mutex互斥量（内核模式同步）**: 基于内核对象的互斥锁，可跨进程使用
  - When to use: 需要跨进程同步，或需要超时防止死锁时
  - How: CreateMutex创建 -> WaitForSingleObject获取 -> ReleaseMutex释放 -> CloseHandle销毁

- **Semaphore信号量（内核模式同步）**: 基于内核对象的计数同步机制
  - When to use: 需要控制同时访问资源的线程数量时
  - How: CreateSemaphore创建 -> WaitForSingleObject获取(减1) -> ReleaseSemaphore释放(加1)

- **Event事件对象（内核模式同步）**: 基于事件的同步机制，可手动或自动设置signaled状态
  - When to use: 需要主动通知线程某事件发生时（如数据就绪通知）
  - How: CreateEvent创建 -> SetEvent设为signaled -> ResetEvent重置 -> WaitForSingleObject等待

## Key Concepts
- **用户模式（User Mode）**: 应用程序运行模式，禁止访问物理设备和操作系统内存
- **内核模式（Kernel Mode）**: 操作系统运行模式，可访问所有内存和硬件
- **CRITICAL_SECTION**: 用户模式同步对象，非内核对象，速度快
- **Mutex**: 互斥量内核对象，auto-reset模式，可跨进程
- **Semaphore**: 信号量内核对象，值大于0为signaled，等于0为non-signaled
- **Event**: 事件内核对象，支持manual-reset和auto-reset两种模式
- **auto-reset**: WaitForSingleObject返回后自动变回non-signaled
- **manual-reset**: 需手动调用ResetEvent重置为non-signaled

## Mental Models
- Use CRITICAL_SECTION when you need fast, simple synchronization within a single process
- Use Kernel Mutex when you need cross-process synchronization or timeout capability
- Use Semaphore when you need to allow N threads (not just 1) to access a resource simultaneously
- Use Event when you need to explicitly signal threads about the occurrence of specific conditions

## Anti-patterns
- **获取Mutex后忘记ReleaseMutex**: 导致死锁，等待该Mutex的线程永久阻塞
- **在用户模式同步需要跨进程时使用CRITICAL_SECTION**: CRITICAL_SECTION是非内核对象，无法跨进程使用
- **将整个循环放入临界区**: 虽然保证正确性，但严重降低并发性能，应仅保护共享数据操作

## Code Examples
```c
/* CRITICAL_SECTION同步示例 */
#include <stdio.h>
#include <windows.h>
#include <process.h>

#define NUM_THREAD 50
unsigned WINAPI threadInc(void *arg);
unsigned WINAPI threadDes(void *arg);

long long num = 0;
CRITICAL_SECTION cs;

int main(int argc, char *argv[])
{
    HANDLE tHandles[NUM_THREAD];
    int i;

    InitializeCriticalSection(&cs);    /* 初始化CS */

    for (i = 0; i < NUM_THREAD; i++) {
        if (i % 2)
            tHandles[i] = (HANDLE)_beginthreadex(NULL, 0, threadInc, NULL, 0, NULL);
        else
            tHandles[i] = (HANDLE)_beginthreadex(NULL, 0, threadDes, NULL, 0, NULL);
    }

    WaitForMultipleObjects(NUM_THREAD, tHandles, TRUE, INFINITE);
    DeleteCriticalSection(&cs);       /* 销毁CS */
    printf("result: %lld \n", num);
    return 0;
}

unsigned WINAPI threadInc(void *arg)
{
    int i;
    EnterCriticalSection(&cs);        /* 进入临界区 */
    for (i = 0; i < 50000000; i++)
        num += 1;
    LeaveCriticalSection(&cs);        /* 离开临界区 */
    return 0;
}

/* Mutex同步示例 */
/* HANDLE hMutex = CreateMutex(NULL, FALSE, NULL);  创建 */
/* WaitForSingleObject(hMutex, INFINITE);            获取 */
/* ... 临界区操作 ...                                  */
/* ReleaseMutex(hMutex);                             释放 */
/* CloseHandle(hMutex);                              销毁 */

/* Semaphore同步示例 */
/* HANDLE semOne = CreateSemaphore(NULL, 0, 1, NULL);  初始值0,non-signaled */
/* HANDLE semTwo = CreateSemaphore(NULL, 1, 1, NULL);  初始值1,signaled */
/* WaitForSingleObject(semTwo, INFINITE);  获取信号量(值减1) */
/* ... 临界区操作 ... */
/* ReleaseSemaphore(semOne, 1, NULL);       释放信号量(值加1) */
```
- **What it demonstrates**: CRITICAL_SECTION的完整使用流程（初始化->进入->操作->离开->销毁），以及Mutex和Semaphore的使用模式

## Reference Tables
| 同步方式 | 模式 | 内核对象 | 跨进程 | 超时支持 | 速度 |
|---------|------|---------|--------|---------|------|
| CRITICAL_SECTION | 用户 | 否 | 否 | 否 | 最快 |
| Mutex | 内核 | 是 | 是 | 是 | 较慢 |
| Semaphore | 内核 | 是 | 是 | 是 | 较慢 |
| Event | 内核 | 是 | 是 | 是 | 较慢 |

| 同步对象 | 创建函数 | 获取函数 | 释放函数 | 销毁函数 |
|---------|---------|---------|---------|---------|
| CRITICAL_SECTION | InitializeCriticalSection | EnterCriticalSection | LeaveCriticalSection | DeleteCriticalSection |
| Mutex | CreateMutex | WaitForSingleObject | ReleaseMutex | CloseHandle |
| Semaphore | CreateSemaphore | WaitForSingleObject | ReleaseSemaphore | CloseHandle |
| Event | CreateEvent | WaitForSingleObject | SetEvent | CloseHandle |

| Mutex状态 | 条件 | 说明 |
|-----------|------|------|
| signaled | 未被任何线程拥有 | WaitForSingleObject直接返回 |
| non-signaled | 被某线程拥有 | 其他线程调用WaitForSingleObject阻塞 |

| Semaphore状态 | 信号量值 | 说明 |
|---------------|---------|------|
| signaled | > 0 | WaitForSingleObject返回，值减1 |
| non-signaled | = 0 | WaitForSingleObject阻塞 |

## Key Takeaways
1. Windows同步分为用户模式（CRITICAL_SECTION）和内核模式（Mutex/Semaphore/Event）两类
2. CRITICAL_SECTION速度最快但功能有限（不能跨进程、不能设超时），适合单进程内高性能同步
3. 内核模式同步通过WaitForSingleObject获取锁（进入signaled检测），通过ReleaseXxx释放锁
4. Mutex是auto-reset模式内核对象，WaitForSingleObject获取时自动变non-signaled，ReleaseMutex释放后变signaled
5. Semaphore通过信号量值控制并发数，值为0时non-signaled，大于0时signaled
6. 内核模式同步可指定超时时间，有效防止死锁导致的永久阻塞

## Connects To
- **Ch 18**: Linux的mutex和semaphore与Windows的Mutex和Semaphore概念对应，但Windows区分用户/内核模式
- **Ch 19**: 本章的同步机制基于第19章的内核对象状态（signaled/non-signaled）理论
- **Ch 21**: 异步通知I/O模型中Event对象的使用建立在本章Event同步机制的基础上
