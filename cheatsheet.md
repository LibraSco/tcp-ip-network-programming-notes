# 速查表 — TCP/IP 网络编程

## 服务器端选择决策

| 场景 | 推荐模型 | 章节 |
|------|---------|------|
| 学习/原型 | 迭代服务器 | Ch 4 |
| 少量连接、Linux | 多进程 (fork) | Ch 10 |
| 中等连接、Linux | select 或 epoll | Ch 12, 17 |
| 高并发、Linux | epoll + 线程池 | Ch 17, 18 |
| 少量连接、Windows | 多线程 | Ch 19 |
| 高并发、Windows | IOCP | Ch 23 |

## TCP vs UDP 决策规则

| 维度 | TCP | UDP |
|------|-----|-----|
| 可靠性 | 可靠（重传、排序、流控） | 不可靠 |
| 连接 | 需要连接（三次握手） | 无连接 |
| 数据边界 | 无边界（流式） | 有边界（数据报） |
| 速度 | 较慢 | 较快 |
| 适用场景 | 文件传输、Web、邮件 | 视频、DNS、多播 |
| 函数 | send/recv 或 read/write | sendto/recvfrom |
| 书中章节 | Ch 4, 5 | Ch 6 |

## 套接字创建三参数速查

| 参数 | 值 | 含义 |
|------|---|------|
| domain | PF_INET / PF_INET6 | IPv4 / IPv6 |
| type | SOCK_STREAM | 面向连接（TCP）|
| type | SOCK_DGRAM | 面向数据报（UDP）|
| protocol | 0 | 自动选择协议 |

## 服务器端四步流程
1. `socket()` — 创建套接字
2. `bind()` — 分配地址（IP + 端口）
3. `listen()` — 转为监听状态
4. `accept()` — 接受连接，返回新套接字

## 客户端两步流程
1. `socket()` — 创建套接字
2. `connect()` — 发起连接（系统自动分配 IP/端口）

## 字节序转换函数

| 函数 | 方向 | 位宽 |
|------|------|------|
| `htons()` | 主机 → 网络 | 16 位 |
| `htonl()` | 主机 → 网络 | 32 位 |
| `ntohs()` | 网络 → 主机 | 16 位 |
| `ntohl()` | 网络 → 主机 | 32 位 |

## 地址转换函数

| 函数 | 功能 | 注意 |
|------|------|------|
| `inet_addr()` | 字符串 → in_addr_t | 废弃，用 inet_aton |
| `inet_aton()` | 字符串 → struct in_addr | 推荐 |
| `inet_ntoa()` | struct in_addr → 字符串 | 静态缓冲区，线程不安全 |
| `inet_pton()` | 字符串 → 二进制 | IPv4/IPv6 通用 |

## 套接字可选项速查

| 选项名 | 层级 | 功能 | 默认 |
|--------|------|------|------|
| `SO_REUSEADDR` | SOL_SOCKET | 解决 Time-wait 端口占用 | 关闭 |
| `SO_SNDBUF` | SOL_SOCKET | 输出缓冲大小 | 系统决定 |
| `SO_RCVBUF` | SOL_SOCKET | 输入缓冲大小 | 系统决定 |
| `TCP_NODELAY` | IPPROTO_TCP | 禁用 Nagle 算法 | 关闭（Nagle 开启）|
| `SO_BROADCAST` | SOL_SOCKET | 启用广播 | 关闭 |
| `IP_MULTICAST_TTL` | IPPROTO_IP | 多播 TTL | 1 |
| `IP_ADD_MEMBERSHIP` | IPPROTO_IP | 加入多播组 | — |

## select 调用流程
1. `FD_ZERO(&read_fds)` — 清空集合
2. `FD_SET(fd, &read_fds)` — 注册关心的 fd
3. `select(maxfd+1, &read_fds, NULL, NULL, &timeout)` — 等待事件
4. `FD_ISSET(fd, &read_fds)` — 检查哪个 fd 有数据

## epoll 调用流程
1. `epoll_create(EPOLL_SIZE)` — 创建 epoll 实例
2. `epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event)` — 注册 fd
3. `epoll_wait(epfd, events, MAX_EVENTS, -1)` — 等待事件
4. 遍历返回的 events 数组处理每个就绪 fd

## 条件触发 vs 边缘触发

| 维度 | 条件触发 (LT) | 边缘触发 (ET) |
|------|-------------|-------------|
| 通知时机 | 只要有数据就通知 | 仅状态变化时通知一次 |
| 读取方式 | 可阻塞读 | 必须非阻塞循环读至 EAGAIN |
| 编程难度 | 简单 | 复杂 |
| 性能 | 较低 | 较高 |
| epoll 默认 | 是 | 否 |

## 线程同步方式对比

| 同步方式 | 平台 | 性能 | 适用场景 |
|---------|------|------|---------|
| mutex (互斥锁) | Linux | 中 | 临界区保护 |
| semaphore (信号量) | Linux | 中 | 资源计数 |
| CRITICAL_SECTION | Windows | 高(用户态) | 轻量临界区 |
| Mutex (内核对象) | Windows | 低(内核态) | 跨进程同步 |
| Event (事件对象) | Windows | 低(内核态) | 条件通知 |

## 半关闭决策规则
- `close()` = 同时关闭收发 → 如果接收缓冲区有数据，对端无法接收
- `shutdown(SHUT_WR)` = 只关闭发送（发 EOF），保留接收能力
- **When to use shutdown**: 文件传输结束后需确认对方已接收完毕 → 发 EOF + 等待对方关闭 → 再 close

## Time-wait 处理规则
- **问题**: 服务器重启后端口被 Time-wait 占用，bind 失败
- **规则**: 服务器端总是设置 `SO_REUSEADDR`，允许地址再分配
- **设置时机**: bind 之前，socket 之后

## Windows I/O 模型选择

| 模型 | 并发能力 | 复杂度 | 适用场景 |
|------|---------|--------|---------|
| 阻塞 + 多线程 | 低 | 低 | 简单场景 |
| WSAAsyncSelect | 中 | 低 | 需窗口消息循环 |
| 重叠 I/O | 高 | 中 | 高性能 I/O |
| IOCP | 最高 | 高 | 大规模并发服务器 |
