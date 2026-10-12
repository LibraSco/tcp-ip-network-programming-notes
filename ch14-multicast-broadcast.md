# Chapter 14: 多播与广播(Multicast & Broadcast)

## Core Idea
多播和广播都是基于 UDP 的"一次性向多个主机发送相同数据"的技术:多播借助路由器复制数据包、用 TTL 控制传输距离、用 D 类组地址向"加入组"的主机传递,可跨网络;广播仅向同一网络的所有主机传输,分直接广播和本地广播两类。

## Frameworks Introduced
- **多播(Multicast)**: 基于 UDP、向加入特定多播组的大量主机同时传递数据的方式;服务器只发 1 次,路由器复制并传递到组内所有主机,组数可任意增加。
  - When to use: 需要向大量客户端发送相同数据(如多媒体实时传输),且希望节省服务器与网络流量时。
  - How: Sender 创建 UDP 套接字 + setsockopt(IP_MULTICAST_TTL) 设 TTL + sendto 多播组地址;Receiver 创建 UDP 套接字 + bind + setsockopt(IP_ADD_MEMBERSHIP) 加入组 + recvfrom。
- **TTL 设置框架**: TTL(Time to Live)决定数据包传输距离,每经一个路由器减 1,为 0 时销毁;通过套接字可选项 `IPPROTO_IP` 层的 `IP_MULTICAST_TTL` 设置。
  - When to use: 任何多播 Sender 发送数据前,必须设置 TTL。
  - How: `int time_live=64; setsockopt(send_sock, IPPROTO_IP, IP_MULTICAST_TTL, (void*)&time_live, sizeof(time_live));`
- **加入多播组框架**: Receiver 通过 `IP_ADD_MEMBERSHIP` 选项加入组,声明接收发往某 D 类地址的数据。
  - When to use: 多播 Receiver 接收数据前,必须加入对应多播组。
  - How: 构造 `struct ip_mreq`(imr_multiaddr=组地址, imr_interface=本机地址或 INADDR_ANY)→ `setsockopt(recv_sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, (void*)&join_adr, sizeof(join_adr));`
- **广播(Broadcast)**: 向同一网络中所有主机传输数据,基于 UDP;按地址形式分直接广播与本地广播,Sender 需开启 `SO_BROADCAST` 选项,Receiver 无需额外设置。
  - When to use: 需向本网络全部主机一次性发送数据(且无需多播组管理)时。
  - How: Sender `setsockopt(SO_BROADCAST, 1)` 开启广播 + sendto 广播地址;Receiver 普通 UDP bind + recvfrom。

## Key Concepts
- **多播(Multicast)**: 基于 UDP,服务器只发 1 次数据,路由器复制数据包并传递到加入组的主机;组数可任意增加,加入特定组即可接收发往该组的数据。
- **多播组**: D 类 IP 地址(224.0.0.0 ~ 239.255.255.255);"加入多播组"即声明接收发往某 D 类地址的数据。
- **D 类 IP 地址**: 224.0.0.0 ~ 239.255.255.255,用于多播组地址。
- **TTL(Time to Live)**: 决定数据包传输距离的整数,每经过 1 个路由器减 1,为 0 时数据包被销毁;设置过大影响网络流量,过小无法到达目标。
- **IP_MULTICAST_TTL**: IPPROTO_IP 层套接字选项,设置多播数据包的 TTL。
- **IP_ADD_MEMBERSHIP**: IPPROTO_IP 层套接字选项,使主机加入指定多播组以接收数据。
- **struct ip_mreq**: `{struct in_addr imr_multiaddr; struct in_addr imr_interface;}`,前者填组地址,后者填加入主机地址(可用 INADDR_ANY)。
- **Sender/Receiver**: 多播中以"发送者(Sender)"和"接收者(Receiver)"替代服务器端/客户端;Sender 是发送主体,Receiver 是加入组的数据接收主体。
- **广播(Broadcast)**: 向同一网络中所有主机传输数据,基于 UDP;与多播区别在于传输范围仅限同一网络。
- **直接广播(Directed Broadcast)**: 地址中网络部分为真实网络地址、主机部分全为 1(如 192.12.34.255),向特定网络所有主机传输。
- **本地广播(Local Broadcast)**: 地址 255.255.255.255,向数据发送主机所在同一网络的所有主机传输。
- **SO_BROADCAST**: SOL_SOCKET 层套接字选项,默认生成的套接字阻止广播,需 setsockopt 设为 1 才能发送广播数据;仅 Sender 需设置。

## Mental Models
- Use **多播** when 需向大量、可能跨网络的主机发送相同数据,且希望服务器只发一次、由路由器复制分发以节省流量。
- Use **IP_MULTICAST_TTL** when 多播 Sender 发送前控制数据传输距离:大值跨更多路由器、小值限制在本地区域。
- Use **IP_ADD_MEMBERSHIP** when Receiver 要接收某多播组数据:加入组后即可收到发往该 D 类地址的数据,组数无限制。
- Use **广播 + SO_BROADCAST** when 仅需向本网络所有主机一次性发数据,无需组管理;Sender 开 SO_BROADCAST、用直接或本地广播地址。
- Use **直接广播地址** when 向特定网络(已知网络地址)所有主机传输;用 **255.255.255.255** when 向本机所在网络全部主机传输。

## Anti-patterns
- **多播 Sender 不设 TTL**: TTL 是多播数据传递的必需项,不设置无法控制传输距离,可能传不到目标或浪费网络流量。
- **Receiver 不加入多播组就想接收**: 多播数据只发给"加入组"的主机,未 IP_ADD_MEMBERSHIP 则收不到数据。
- **广播未开 SO_BROADCAST 就发数据**: 默认套接字阻止广播,不 setsockopt(SO_BROADCAST,1) 会发送失败。
- **混淆多播与广播范围**: 多播可跨网络(只要加入组),广播仅限同一网络;误用广播替代多播会导致跨网主机收不到。
- **误以为多播 Sender/Receiver 有连接关系**: 多播属广播范畴,无 TCP 连接状态,运行顺序不重要,但延迟运行 Receiver 会错过之前已发的数据。

## Code Examples
```c
/* news_sender.c —— 多播 Sender:设 TTL 后向组地址发送 */
#define TTL 64
int send_sock;
struct sockaddr_in mul_adr;
int time_live=TTL;

send_sock=socket(PF_INET, SOCK_DGRAM, 0);     /* 多播基于 UDP */
memset(&mul_adr, 0, sizeof(mul_adr));
mul_adr.sin_family=AF_INET;
mul_adr.sin_addr.s_addr=inet_addr(argv[1]);   /* 多播组地址(D 类) */
mul_adr.sin_port=htons(atoi(argv[2]));

setsockopt(send_sock, IPPROTO_IP,
    IP_MULTICAST_TTL, (void*)&time_live, sizeof(time_live));  /* 设 TTL */

while(!feof(fp))   /* Broadcasting */
{
    fgets(buf, BUF_SIZE, fp);
    sendto(send_sock, buf, strlen(buf), 0,
           (struct sockaddr*)&mul_adr, sizeof(mul_adr));
    sleep(2);
}
```
```c
/* news_receiver.c —— 多播 Receiver:加入组后接收 */
int recv_sock;
struct sockaddr_in adr;
struct ip_mreq join_adr;

recv_sock=socket(PF_INET, SOCK_DGRAM, 0);
memset(&adr, 0, sizeof(adr));
adr.sin_family=AF_INET;
adr.sin_addr.s_addr=htonl(INADDR_ANY);
adr.sin_port=htons(atoi(argv[2]));
bind(recv_sock, (struct sockaddr*)&adr, sizeof(adr));

join_adr.imr_multiaddr.s_addr=inet_addr(argv[1]);  /* 多播组地址 */
join_adr.imr_interface.s_addr=htonl(INADDR_ANY);   /* 本机地址 */

setsockopt(recv_sock, IPPROTO_IP,
    IP_ADD_MEMBERSHIP, (void*)&join_adr, sizeof(join_adr));  /* 加入组 */

while(1)
{
    str_len=recvfrom(recv_sock, buf, BUF_SIZE-1, 0, NULL, 0);
    if(str_len<0) break;
    buf[str_len]=0;
    fputs(buf, stdout);
}
```
```c
/* news_sender_brd.c —— 广播 Sender:开 SO_BROADCAST 后发送 */
int send_sock=socket(PF_INET, SOCK_DGRAM, 0);
int so_brd=1;
/* broad_adr.sin_addr 设为广播地址(直接/本地) */
setsockopt(send_sock, SOL_SOCKET,
    SO_BROADCAST, (void*)&so_brd, sizeof(so_brd));   /* 关键:开启广播 */
sendto(send_sock, buf, strlen(buf), 0,
       (struct sockaddr*)&broad_adr, sizeof(broad_adr));
/* Receiver 无需额外设置,普通 UDP bind + recvfrom 即可 */
```
- **What it demonstrates**: 多播 Sender 设 IP_MULTICAST_TTL 后 sendto 组地址;Receiver 用 IP_ADD_MEMBERSHIP 加入组后 recvfrom;广播 Sender 用 SO_BROADCAST 开启后发广播地址。

## Reference Tables

| 概念 | 多播(Multicast) | 广播(Broadcast) |
|------|-----------------|------------------|
| 传输范围 | 可跨网络(加入组即可) | 仅同一网络 |
| 地址类型 | D 类(224.0.0.0~239.255.255.255) | 直接广播(网络地址+全1主机) / 本地 255.255.255.255 |
| Sender 必设选项 | IP_MULTICAST_TTL | SO_BROADCAST |
| Receiver 必设选项 | IP_ADD_MEMBERSHIP | 无(普通 UDP) |
| 数据复制方 | 路由器复制数据包 | — |
| 协议基础 | UDP | UDP |

| 广播类型 | 地址形式 | 说明 |
|---------|---------|------|
| 直接广播(Directed) | 网络地址 + 主机部分全 1(如 192.12.34.255) | 向特定网络所有主机传输 |
| 本地广播(Local) | 255.255.255.255 | 向发送方所在网络所有主机传输 |

| 套接字选项 | 协议层 | 作用 |
|-----------|--------|------|
| IP_MULTICAST_TTL | IPPROTO_IP | 设置多播数据包 TTL |
| IP_ADD_MEMBERSHIP | IPPROTO_IP | 加入多播组 |
| SO_BROADCAST | SOL_SOCKET | 开启套接字广播能力(默认禁止) |

| struct ip_mreq 成员 | 含义 |
|---------------------|------|
| imr_multiaddr | 加入的多播组地址(D 类) |
| imr_interface | 加入主机地址(可用 INADDR_ANY) |

## Key Takeaways
1. 多播和广播都基于 UDP、都是"一次向多个主机发相同数据":多播靠路由器复制、用 D 类组地址、可跨网络;广播仅限同一网络。
2. 多播 Sender 三步:创建 UDP 套接字 → setsockopt(IP_MULTICAST_TTL) 设传输距离 → sendto 多播组地址;Receiver 三步:UDP+bind → setsockopt(IP_ADD_MEMBERSHIP) 加入组 → recvfrom。
3. TTL 每经路由器减 1,为 0 销毁;设大影响流量、设小传不到目标,需平衡——多播 Sender 必须设置 TTL。
4. 多播组是 D 类地址(224.0.0.0~239.255.255.255),加入组即声明接收该组数据,组数无限制;多播无连接状态,运行顺序不重要,但迟到的 Receiver 会错过已发数据。
5. 广播分两类:直接广播(网络地址+主机全 1)向特定网络、本地广播(255.255.255.255)向本网络所有主机;默认套接字禁止广播,Sender 必须 setsockopt(SO_BROADCAST,1)。
6. 广播与多播在数据通信角度的区别主要在 IP 地址形式与传输范围;Receiver 实现几乎与普通 UDP 相同,无需额外设置。

## Connects To
- **Ch 5/6(UDP)**: 多播与广播都基于 UDP,套接字创建与 sendto/recvfrom 用法源自 UDP 基础章节。
- **Ch 9(套接字可选项)**: TTL、加入组、开启广播都通过 setsockopt 套接字可选项完成,沿用第 9 章 setsockopt 机制。
- **Ch 13(send/recv)**: 多播用 sendto/recvfrom,广播同;与第 13 章的 I/O 函数族同属数据收发体系。
- **IP 多播标准(RFC 1112)**: 多播基于 IGMP/D 类地址,MBone 虚拟网络承载;TTL 是 IP 头标准字段。
