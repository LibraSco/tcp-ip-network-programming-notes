# Chapter 5: 基于TCP的服务器端/客户端(2)

## Core Idea
回声客户端存在数据边界问题导致可能的错误接收,解决方法是按发送数据的长度精确接收;应用层协议设计是网络编程的核心,计算器示例演示了如何设计协议并在单次数组中打包多种类型数据传输。

## Frameworks Introduced
- **精确长度接收框架**: TCP不存在数据边界,一次write的数据可能被多次read返回,因此接收端必须循环read直到累计接收字节数达到发送长度。
  - When to use: TCP客户端需要确保完整接收服务器发送的指定长度数据时
  - How: 记录已接收字节数(recv_len),循环read直到recv_len >= 期望接收长度

- **应用层协议设计框架**: 在编写网络程序前先设计数据交换规则(协议),包括数据类型、长度、顺序、分隔方式等,协议设计完成相当于实现了一半程序。
  - When to use: 所有需要客户端与服务器端交换结构化数据的场景
  - How: 定义每个字段的字节大小和传输顺序 -> 客户端按协议打包 -> 服务器端按协议解析

- **多类型数据打包框架**: 将不同类型数据(int、char)存入char数组,通过指针类型转换实现混合打包,一次性write传输。
  - When to use: 需要在一次传输中发送多种类型数据时
  - How: 声明char数组 -> 用(char)存1字节数据 -> 用(int*)存4字节数据 -> 一次性write

## Key Concepts
- **回声客户端实现问题**: 简单的echo_client每次write后立即read,但TCP无数据边界,可能read到不完整或多余数据
- **应用层协议**: 程序员定义的服务器端与客户端之间的数据传输规则
- **TCP的I/O缓冲**: 套接字内部的输入/输出缓冲,数据先进入缓冲再传输,read/write调用次数无直接意义
- **数据边界**: TCP中不存在数据边界,3次write的100字节可能被1次read全部读取
- **计算器协议**: 客户端传递算数个数(1字节)+操作数(每个4字节)+运算符(1字节),服务器返回结果(4字节)
- **EOF传递**: 调用close会向对方传递EOF,可用于通知数据传输结束

## Mental Models
- Use "accumulate until expected length" when receiving TCP data: loop read until recv_len >= expected, because one read may not return all data
- Use "design the protocol first, then code" when building network applications: specifying byte sizes, types, and order of each field is half the implementation
- Use "char array as a universal container" when packing mixed-type data: store 1-byte data directly, cast to int* for 4-byte data, send all at once with write

## Anti-patterns
- **回声客户端假设一次read返回所有数据**: TCP无数据边界,write一次的数据可能需要多次read才能完整接收——简单echo_client在高负载时可能出错
- **未设计协议直接编码**: 没有先设计数据交换规则就开始写代码,会导致客户端和服务器端数据格式不匹配,难以调试
- **用特定字符作为数据结束标记**: 约定特殊字符代表传输结束会与文件内容冲突——应使用EOF(通过close或shutdown)通知结束

## Code Examples
```c
// op_client.c —— 计算器客户端(应用层协议+多类型打包)
#define BUF_SIZE 1024
#define RLT_SIZE 4     // 运算结果字节数
#define OPSZ 4         // 操作数字节数

int sock;
char opmsg[BUF_SIZE];   // 用char数组打包所有数据
int result, opnd_cnt, i;
struct sockaddr_in serv_adr;

sock = socket(PF_INET, SOCK_STREAM, 0);
// ... connect ...

fputs("Operand count: ", stdout);
scanf("%d", &opnd_cnt);
opmsg[0] = (char)opnd_cnt;           // 第1字节:算数个数(1字节整数)

for (i = 0; i < opnd_cnt; i++)
{
    printf("Operand %d: ", i + 1);
    scanf("%d", (int*)&opmsg[i*OPSZ+1]);  // 每4字节存一个操作数(int*转换)
}
fgetc(stdin);                         // 清除缓冲区中的\n
fputs("Operator: ", stdout);
scanf("%c", &opmsg[opnd_cnt*OPSZ+1]); // 最后1字节:运算符

write(sock, opmsg, opnd_cnt*OPSZ+2);  // 一次性传输所有数据
read(sock, &result, RLT_SIZE);         // 接收4字节运算结果
printf("Operation result: %d \n", result);
close(sock);
```
- **What it demonstrates**: 应用层协议设计(1字节个数+4字节操作数*个数+1字节运算符)和多类型数据打包到char数组

```c
// op_server.c —— 计算器服务器(精确长度接收)
clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
read(clnt_sock, &opnd_cnt, 1);        // 先接收1字节:算数个数

recv_len = 0;
while ((opnd_cnt*OPSZ+1) > recv_len)  // 循环直到接收完所有操作数+运算符
{
    recv_cnt = read(clnt_sock, &opinfo[recv_len], BUF_SIZE-1);
    recv_len += recv_cnt;
}
result = calculate(opnd_cnt, (int*)opinfo, opinfo[recv_len-1]);
write(clnt_sock, (char*)&result, sizeof(result));  // 返回4字节结果
close(clnt_sock);
```
- **What it demonstrates**: 服务器端按协议精确接收指定长度数据,而非假设一次read返回全部

## Reference Tables

| 计算器协议字段 | 类型 | 字节数 | 说明 |
|---------------|------|--------|------|
| 算数个数 | char | 1 | 客户端->服务器,第一个字节 |
| 操作数 | int | 4*个数 | 客户端->服务器,紧随个数 |
| 运算符 | char | 1 | +、-、* 之一 |
| 运算结果 | int | 4 | 服务器->客户端 |

| 回声客户端问题 | 原因 | 解决方案 |
|---------------|------|---------|
| 数据接收不完整 | TCP无数据边界,一次read可能返回部分 | 循环read直到接收完期望长度 |
| 一次性传输多种类型 | 不同类型数据需混合打包 | 用char数组+指针转换统一存储 |

## Key Takeaways
1. TCP不存在数据边界,write和read的调用次数无直接对应关系,接收端必须按期望长度循环接收
2. 应用层协议设计是网络编程的核心——先定义数据格式(类型、字节大小、顺序),协议设计完成等于实现了一半
3. 多种类型数据可通过char数组打包,用(char)存1字节数据,用(int*)存4字节数据,一次性write传输
4. 服务器端接收数据时应循环read累计recv_len直到达到期望长度,不能假设一次read返回全部
5. 调用close会向对方传递EOF,可用于通知数据传输结束——但close同时关闭输入输出流(第7章讲解半关闭)

## Connects To
- **Ch 4**: 本章基于第4章的回声服务器/客户端,解决其"不完美实现"问题
- **Ch 2**: TCP无数据边界的特性在第2章介绍,本章给出实际问题和解决方案
- **Ch 7**: close传递EOF但同时关闭两个流的问题在第7章通过半关闭(shutdown)解决
- **Ch 9**: TCP的I/O缓冲和流控制在第9章作为套接字选项的基础理论补充
