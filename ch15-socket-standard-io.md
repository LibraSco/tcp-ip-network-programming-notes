# Chapter 15: 套接字与标准I/O

## Core Idea
标准I/O函数（fopen/fputs/fgets等）提供缓冲机制和跨平台可移植性，通过fdopen可将套接字文件描述符转换为FILE指针，从而利用标准I/O的优势进行网络数据传输。

## Frameworks Introduced
- **标准I/O（Standard I/O）**: 基于C标准库的I/O函数，通过FILE指针操作数据流，内部自带缓冲区
  - When to use: 需要缓冲功能、跨平台兼容性、或简化数据格式化操作时
  - How: 用fdopen将文件描述符包装为FILE指针，再使用fputs/fgets等函数

- **fdopen/fileno转换框架**: 在文件描述符与FILE指针之间双向转换的机制
  - When to use: 已有套接字文件描述符，但想使用标准I/O函数时
  - How: fdopen(fd, mode)将fd转为FILE*；fileno(FILE*)将FILE*转为fd

## Key Concepts
- **FILE指针**: 标准I/O函数操作的数据流句柄，内部包含缓冲区和位置信息
- **fdopen**: 将文件描述符转换为FILE指针的函数，是连接系统级I/O与标准I/O的桥梁
- **fileno**: 将FILE指针转换为文件描述符的函数
- **缓冲（Buffering）**: 标准I/O在用户空间维护的数据暂存区，减少系统调用次数
- **全缓冲（Fully Buffered）**: 数据填满缓冲区后才进行实际I/O操作的缓冲模式
- **行缓冲（Line Buffered）**: 遇到换行符时进行I/O操作的缓冲模式
- **fflush**: 主动将缓冲区数据刷新到实际I/O的函数

## Mental Models
- Use fdopen when you have a socket fd but want buffering and formatted I/O
- Use fileno when you need to pass a FILE* to a function expecting a file descriptor
- Use fflush when buffered data must be transmitted immediately
- Use fclose when you want to close both the FILE stream and underlying file descriptor simultaneously

## Anti-patterns
- **混用文件描述符I/O与标准I/O对同一套接字操作**: 会导致数据顺序混乱，因为标准I/O有缓冲而文件描述符I/O没有
- **fdopen后忘记fclose**: fclose会同时关闭FILE指针和底层文件描述符，仅close文件描述符会导致资源泄漏

## Code Examples
```c
/* 将套接字文件描述符转换为FILE指针，使用标准I/O进行读写 */
FILE *readfp = fdopen(clnt_sock, "r");
FILE *writefp = fdopen(clnt_sock, "w");

/* 使用标准I/O函数进行数据传输 */
fputs("FROM SERVER: Hi~ client? \n", writefp);
fflush(writefp);  /* 确保缓冲区数据立即发送 */

fgets(message, BUF_SIZE, readfp);  /* 读取客户端消息 */
fputs(message, writefp);
fflush(writefp);

/* 关闭FILE指针（同时关闭底层文件描述符） */
fclose(readfp);
fclose(writefp);
```
- **What it demonstrates**: 通过fdopen将套接字包装为FILE指针后，可使用fputs/fgets等标准I/O函数进行网络通信，利用标准I/O的缓冲机制提升性能

## Reference Tables
| 函数 | 头文件 | 作用 | 返回值 |
|------|--------|------|--------|
| fdopen(int fd, const char *mode) | <stdio.h> | 将文件描述符转为FILE指针 | 成功FILE*，失败NULL |
| fileno(FILE *stream) | <stdio.h> | 将FILE指针转为文件描述符 | 文件描述符 |
| fflush(FILE *stream) | <stdio.h> | 刷新缓冲区 | 成功0，失败EOF |
| fopen(const char *path, const char *mode) | <stdio.h> | 打开文件并返回FILE指针 | 成功FILE*，失败NULL |
| fputs(const char *s, FILE *stream) | <stdio.h> | 向流写入字符串 | 成功非负，失败EOF |
| fgets(char *s, int size, FILE *stream) | <stdio.h> | 从流读取字符串 | 成功s，失败NULL |

| 缓冲模式 | 说明 | 适用场景 |
|----------|------|----------|
| 全缓冲 | 缓冲区满后才I/O | 文件I/O |
| 行缓冲 | 遇换行符后I/O | 终端I/O |
| 无缓冲 | 不缓冲，立即I/O | 标准错误 |

## Key Takeaways
1. 标准I/O函数通过内部缓冲机制减少系统调用次数，可提升网络I/O性能
2. fdopen是连接套接字（文件描述符）与标准I/O（FILE指针）的关键函数
3. fclose会同时关闭FILE指针和底层文件描述符，无需额外close
4. 使用标准I/O进行套接字编程时，fflush确保数据及时发送，避免缓冲区滞留
5. 标准I/O提供了比read/write更好的可移植性，同一代码可在不同平台编译运行

## Connects To
- **Ch 16**: fdopen创建的FILE指针在I/O流分离中扮演核心角色，通过dup复制文件描述符实现读写分离
- **Ch 24**: Linux版Web服务器实现中使用fdopen和dup结合标准I/O处理HTTP请求
- **C标准库**: fdopen/fileno是POSIX标准定义的函数，连接C标准库与系统级I/O
