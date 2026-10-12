# Chapter 3: 地址族与数据序列

## Core Idea
网络地址信息通过sockaddr_in结构体保存,其中端口号和IP地址必须以网络字节序(大端序)存储;使用inet_addr/inet_aton/inet_ntoa进行字符串与整数地址间的转换,使用htons/htonl进行字节序转换。

## Frameworks Introduced
- **地址信息结构体框架(sockaddr_in)**: 用结构体sockaddr_in替代原始sockaddr来填充IPv4地址信息,因为sockaddr_in的成员分工明确(sin_family/sin_port/sin_addr/sin_zero),填充更方便,最后强制转换为sockaddr*传递给bind。
  - When to use: 给套接字分配地址(服务器端bind)或准备连接目标地址(客户端connect)
  - How: memset清零 -> 设sin_family=AF_INET -> 设sin_addr(用inet_addr或INADDR_ANY) -> 设sin_port(用htons) -> 传递给bind/connect

- **网络字节序转换框架**: 不同CPU字节序不同(大端/小端),网络传输统一使用大端序(网络字节序),通过htons/htonl/ntohs/ntohl函数完成主机序与网络序间的转换。
  - When to use: 向sockaddr_in填充端口号和IP地址时(必须转为网络字节序)
  - How: 端口号用htons(16位),IP地址用htonl(32位);接收到的网络数据用ntohs/ntohl转回主机序

- **IP地址字符串转换框架**: 程序员熟悉点分十进制字符串(如"211.214.107.99"),但结构体需要32位整数,用inet_addr/inet_aton/inet_ntoa完成互转。
  - When to use: 将用户输入的IP字符串填入结构体,或将结构体中的IP输出为字符串
  - How: 字符串->整数用inet_addr或inet_aton;整数->字符串用inet_ntoa

## Key Concepts
- **sockaddr_in结构体**: 保存IPv4地址信息的结构体,含sin_family(地址族)、sin_port(端口号)、sin_addr(IP地址)、sin_zero[8](填充)
- **sockaddr结构体**: 通用地址结构体,含sin_family和sa_data[14],因填充不便而被sockaddr_in替代
- **网络字节序(Network Byte Order)**: 统一为大端序(Big-Endian)的网络数据传输标准
- **大端序(Big-Endian)**: 高位字节存放在低位地址的排列方式
- **小端序(Little-Endian)**: 高位字节存放在高位地址的排列方式(多数Intel CPU)
- **htons**: host to network short,将16位主机字节序转为网络字节序
- **htonl**: host to network long,将32位主机字节序转为网络字节序
- **inet_addr**: 将点分十进制IP字符串转为32位网络字节序整数,失败返回INADDR_NONE
- **inet_aton**: 功能同inet_addr,但将结果直接填入in_addr结构体,返回1(成功)/0(失败)
- **inet_ntoa**: 将32位网络字节序整数转为点分十进制字符串,返回的字符串存储在内部静态内存中(需立即复制)
- **INADDR_ANY**: 常数,用于自动获取服务器端计算机的IP地址,无需手动输入

## Mental Models
- Use "sockaddr_in is the programmer-friendly version of sockaddr" when filling address info: fill sockaddr_in fields individually, then cast to sockaddr* for bind
- Use "network byte order is always Big-Endian" when deciding whether to convert: anything going into sockaddr_in (port and IP) must be network order
- Use "inet_ntoa returns a static buffer" when handling converted IP strings: copy it immediately with strcpy before calling inet_ntoa again, or the previous result will be overwritten
- Use "INADDR_ANY for servers, explicit IP for clients" when initializing addresses: servers should auto-bind all NICs, clients specify the server's IP

## Anti-patterns
- **直接用主机字节序填充sin_port/sin_addr**: 端口号和IP地址必须以网络字节序(大端序)保存,否则通信会出错——必须用htons/htonl转换
- **未初始化sin_zero**: sockaddr_in中的sin_zero必须填充为0(用memset清零整个结构体),否则bind可能产生异常
- **inet_ntoa返回值长期保存**: inet_ntoa返回的字符串存在内部静态内存,再次调用会覆盖前一次结果——需要保存时必须立即strcpy到自己的缓冲区

## Code Examples
```c
// 网络地址初始化的标准方法
struct sockaddr_in addr;
char *serv_ip = "211.217.168.13";     // 声明IP地址字符串
char *serv_port = "9190";             // 声明端口号字符串

memset(&addr, 0, sizeof(addr));                    // 结构体所有成员初始化为0
addr.sin_family = AF_INET;                         // 指定地址族
addr.sin_addr.s_addr = inet_addr(serv_ip);         // 基于字符串的IP地址初始化
addr.sin_port = htons(atoi(serv_port));            // 基于字符串的端口号初始化

// 服务器端使用INADDR_ANY
struct sockaddr_in serv_addr;
char *serv_port = "9190";
memset(&serv_addr, 0, sizeof(serv_addr));
serv_addr.sin_family = AF_INET;
serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);     // 自动获取服务器端IP
serv_addr.sin_port = htons(atoi(serv_port));
```
- **What it demonstrates**: 客户端用inet_addr指定服务器IP,服务器端用INADDR_ANY自动绑定本地IP

```c
// inet_ntoa的使用 —— 注意静态内存覆盖问题
struct sockaddr_in addr1, addr2;
char *str_ptr;
char str_arr[20];

addr1.sin_addr.s_addr = htonl(0x1020304);
addr2.sin_addr.s_addr = htonl(0x1010101);

str_ptr = inet_ntoa(addr1.sin_addr);
strcpy(str_arr, str_ptr);                           // 立即复制,否则会被覆盖
printf("Dotted-Decimal notation1: %s \n", str_ptr);

inet_ntoa(addr2.sin_addr);                          // 再次调用,覆盖了之前的结果
printf("Dotted-Decimal notation2: %s \n", str_ptr); // 输出的是addr2的地址
printf("Dotted-Decimal notation3: %s \n", str_arr); // 输出的是addr1的地址(已复制)
```
- **What it demonstrates**: inet_ntoa返回内部静态内存,再次调用会覆盖,必须立即strcpy保存

## Reference Tables

| sockaddr_in成员 | 类型 | 说明 | 字节序要求 |
|----------------|------|------|-----------|
| sin_family | sa_family_t | 地址族(如AF_INET) | 无需转换 |
| sin_port | uint16_t | 16位TCP/UDP端口号 | 网络字节序 |
| sin_addr | struct in_addr | 32位IP地址 | 网络字节序 |
| sin_zero[8] | char | 不使用,填充为0 | 填充0 |

| 地址族(Address Family) | 含义 |
|------------------------|------|
| AF_INET | IPv4网络协议中使用的地址族 |
| AF_INET6 | IPv6网络协议中使用的地址族 |
| AF_LOCAL | 本地通信中采用的UNIX协议的地址族 |

| 转换函数 | 方向 | 位宽 | 说明 |
|---------|------|------|------|
| htons | host -> network | 16位 | 端口号转换 |
| ntohs | network -> host | 16位 | 端口号反转 |
| htonl | host -> network | 32位 | IP地址转换 |
| ntohl | network -> host | 32位 | IP地址反转 |

| 函数 | 功能 | 返回值 | 特点 |
|------|------|--------|------|
| inet_addr | 字符串IP -> 32位整数 | in_addr_t / INADDR_NONE | 不填入结构体 |
| inet_aton | 字符串IP -> in_addr | 1(成功)/0(失败) | 直接填入结构体,使用频率更高 |
| inet_ntoa | 32位整数 -> 字符串IP | char*(字符串地址) | 返回内部静态内存,需立即复制 |

## Key Takeaways
1. 向sockaddr_in填充数据时,端口号用htons转换,IP地址用htonl或inet_addr(内置转换),sin_family无需转换
2. sockaddr_in是为了方便填充IPv4地址而设计的,最终传递给bind时需强制转换为struct sockaddr*
3. memset清零整个sockaddr_in结构体可同时初始化sin_zero为0,这是标准做法
4. 服务器端优先使用INADDR_ANY自动绑定本机IP,即使多宿主机也能从不同NIC接收数据;客户端需明确指定服务器IP
5. inet_ntoa返回的字符串存储在内部静态内存,再次调用会覆盖——需要保存时必须立即strcpy

## Connects To
- **Ch 1**: hello_server.c和hello_client.c中使用的sockaddr_in在本章详细讲解
- **Ch 4**: TCP服务器/客户端的地址初始化基于本章的sockaddr_in和INADDR_ANY
- **Ch 6**: UDP服务器/客户端同样使用sockaddr_in进行地址初始化
- **POSIX标准**: uint16_t/in_addr_t等数据类型由POSIX标准定义,保证可移植性
