# Chapter 8: 域名及网络地址

## Core Idea
IP地址变更概率高于域名,程序中应使用域名而非硬编码IP地址;通过gethostbyname将域名转为IP地址(返回hostent结构体),通过gethostbyaddr将IP地址转为域名,DNS系统是层次化管理的分布式数据库。

## Frameworks Introduced
- **域名优先框架**: IP地址可能因ISP变更等因素而改变,而域名一旦注册通常不变;程序运行时通过域名获取IP再连接服务器,避免对IP地址的硬编码依赖。
  - When to use: 客户端程序需要连接服务器且IP可能变更时
  - How: 程序中存储域名 -> 运行时调用gethostbyname获取IP -> 用IP连接服务器

- **DNS层次化解析框架**: DNS是层次化管理的分布式数据库,默认DNS服务器无法解析时逐级向上询问,直到根DNS服务器,再向下传递解析结果。
  - When to use: 理解域名到IP地址的转换过程
  - How: 主机请求 -> 默认DNS -> 上级DNS -> 根DNS -> 下级DNS返回IP -> 原路返回

- **hostent结构体解析框架**: gethostbyname返回hostent结构体,包含官方域名、别名列表、地址类型、地址长度和IP地址列表;h_addr_list实际指向in_addr结构体(非字符串),需类型转换。
  - When to use: 调用gethostbyname后提取IP地址信息
  - How: host->h_addr_list[i]是char* -> 强制转换为(struct in_addr*) -> 用inet_ntoa转为字符串

## Key Concepts
- **DNS(Domain Name System)**: 层次化管理的分布式数据库系统,负责域名与IP地址间的转换
- **域名(Domain Name)**: 替代IP地址的人类可读地址,变更概率低于IP地址
- **gethostbyname**: `struct hostent * gethostbyname(const char *hostname)`,将域名转为IP地址,返回hostent结构体
- **gethostbyaddr**: 将IP地址转为域名信息,返回hostent结构体
- **hostent结构体**: 包含h_name(官方域名)、h_aliases(别名列表)、h_addrtype(地址类型)、h_length(地址长度)、h_addr_list(IP地址列表)
- **h_name**: official domain name,官方域名
- **h_aliases**: alias list,同一IP可绑定多个域名,除官方域名外的其他域名
- **h_addrtype**: host address type,IPv4为AF_INET,IPv6为AF_INET6
- **h_length**: address length,IPv4为4字节,IPv6为16字节
- **h_addr_list**: address list,以整数数组形式保存域名对应的IP地址(实际指向in_addr结构体)

## Mental Models
- Use "domain names are more stable than IPs" when deciding what to hardcode in client programs: IPs change with ISPs, domain names persist once registered
- Use "DNS is a hierarchical distributed database" when understanding name resolution: queries bubble up to root, then answers trickle down
- Use "h_addr_list points to in_addr, not strings" when extracting IPs from hostent: cast char* to struct in_addr* before calling inet_ntoa

## Anti-patterns
- **硬编码IP地址到程序代码**: IP地址可能因ISP变更而改变,导致程序失效需要重新编译——应使用域名运行时获取IP
- **直接将h_addr_list当作字符串输出**: h_addr_list中的元素实际指向in_addr结构体而非字符串,直接用%s输出会出错——必须先转为in_addr*再调用inet_ntoa
- **忽略同一域名多个IP的情况**: 高流量网站可能通过多个IP进行负载均衡,h_addr_list可能含多个IP——应遍历列表

## Code Examples
```c
// gethostbyname.c —— 通过域名获取IP地址
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>

int main(int argc, char *argv[])
{
    int i;
    struct hostent *host;

    if (argc != 2) {
        printf("Usage : %s <addr>\n", argv[0]);
        exit(1);
    }

    host = gethostbyname(argv[1]);    // 传入域名
    if (!host)
        error_handling("gethost... error");

    printf("Official name: %s \n", host->h_name);          // 官方域名
    for (i = 0; host->h_aliases[i]; i++)
        printf("Aliases %d: %s \n", i+1, host->h_aliases[i]);  // 别名列表
    printf("Address type: %s \n",
           (host->h_addrtype==AF_INET)?"AF_INET":"AF_INET6");  // 地址类型
    for (i = 0; host->h_addr_list[i]; i++)
        printf("IP addr %d: %s \n", i+1,
               inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));  // IP地址列表
    return 0;
}
```
- **What it demonstrates**: gethostbyname的使用及hostent结构体各成员的遍历输出——注意h_addr_list需转为in_addr*再调用inet_ntoa

```c
// hostent结构体定义
struct hostent
{
    char *h_name;          // official name (官方域名)
    char **h_aliases;      // alias list (别名列表)
    int h_addrtype;        // host address type (地址类型)
    int h_length;          // address length (地址长度)
    char **h_addr_list;    // address list (IP地址列表)
};
```
- **What it demonstrates**: hostent结构体的定义,各成员的含义和用途

## Reference Tables

| hostent成员 | 类型 | 说明 | IPv4典型值 |
|------------|------|------|-----------|
| h_name | char* | 官方域名 | 如www.g.naver.com |
| h_aliases | char** | 别名列表(以NULL结尾) | 如www.naver.com |
| h_addrtype | int | 地址族类型 | AF_INET |
| h_length | int | 地址长度(字节) | 4 |
| h_addr_list | char** | IP地址列表(以NULL结尾) | 指向in_addr结构体 |

| 函数 | 头文件 | 功能 | 返回值 |
|------|--------|------|--------|
| gethostbyname | netdb.h | 域名 -> IP地址 | hostent* / NULL |
| gethostbyaddr | netdb.h | IP地址 -> 域名 | hostent* / NULL |

| DNS解析流程 | 步骤 |
|------------|------|
| 1 | 主机向默认DNS服务器发送域名解析请求 |
| 2 | 默认DNS无法解析时向上级DNS询问 |
| 3 | 逐级向上传递直到根DNS服务器 |
| 4 | 根DNS知道该向哪个下级DNS询问 |
| 5 | 向下传递解析请求,获取IP地址 |
| 6 | IP地址原路返回到发起请求的主机 |

## Key Takeaways
1. IP地址变更概率高于域名,程序应使用域名而非硬编码IP——运行时通过gethostbyname获取IP再连接
2. gethostbyname返回hostent结构体,包含官方域名、别名、地址类型、地址长度和IP地址列表
3. h_addr_list中的元素实际指向in_addr结构体(非字符串),输出IP时需转为(struct in_addr*)再调用inet_ntoa
4. 同一域名可能对应多个IP(负载均衡),h_addr_list以NULL结尾的数组保存所有IP
5. DNS是层次化管理的分布式数据库,无法解析时逐级向上询问直到根DNS服务器

## Connects To
- **Ch 3**: inet_ntoa和in_addr结构体在第3章讲解,本章结合hostent使用
- **Ch 4**: TCP客户端可用gethostbyname获取服务器IP后调用connect
- **Ch 6**: UDP客户端同样可通过域名获取服务器IP地址
- **RFC 1035**: DNS协议的标准定义
