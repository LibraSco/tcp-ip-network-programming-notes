# Chapter 24: 制作HTTP服务器端

## Core Idea
HTTP是无状态（Stateless）的应用层协议，服务器端响应请求后立即断开连接；通过解析HTTP请求消息的请求行（GET方法+文件名），构造响应消息（状态行+头信息+消息体），即可实现简单的Web服务器。

## Frameworks Introduced
- **HTTP协议**: 基于TCP的超文本传输协议，请求-响应后立即断开连接
  - When to use: 实现Web服务器端或理解浏览器与服务器之间的通信机制
  - How: 解析请求行获取方法和文件名 -> 读取文件 -> 构造响应头 -> 发送状态行+头+文件数据

- **多线程Web服务器模型**: 每个客户端请求创建一个线程处理HTTP请求
  - When to use: 简单的Web服务器实现（IOCP/epoll因连接短暂无法发挥优势）
  - How: accept获取连接 -> _beginthreadex/pthread_create创建线程 -> 线程内解析HTTP并响应

## Key Concepts
- **HTTP（Hypertext Transfer Protocol）**: 超文本传输协议，基于TCP的应用层协议
- **Stateless（无状态）**: 服务器不维护客户端状态，每次请求独立处理
- **请求消息（Request Message）**: 客户端发送的消息，由请求行、消息头、消息体组成
- **响应消息（Response Message）**: 服务器返回的消息，由状态行、头信息、消息体组成
- **请求行（Request Line）**: 请求消息第一行，格式为"GET /index.html HTTP/1.1"
- **GET/POST**: HTTP请求方法，GET请求数据，POST传输数据
- **状态码（Status Code）**: 表示请求处理结果的数字，如200(OK)、404(Not Found)、400(Bad Request)
- **Content-type**: 响应头中标识数据类型的字段，如text/html
- **Content-length**: 响应头中标识数据长度的字段

## Mental Models
- Use HTTP protocol when building a web server that serves HTML files to browsers
- Use GET method when the client requests data (file retrieval); use POST when sending data to server
- Use multithreading (not IOCP/epoll) for HTTP servers since connections are too short-lived to benefit from async I/O
- Use fdopen+dup (Linux) or recv+send (Windows) to parse HTTP requests and send responses

## Anti-patterns
- **在HTTP服务器上使用IOCP/epoll期望大幅提升性能**: HTTP是Stateless协议，1次请求-响应后即断开连接，IOCP/epoll的优势无法发挥
- **忽略请求行解析**: 请求行只占1行，是最容易解析的部分，但也是决定响应内容的关键
- **不检查请求方法**: 只实现GET方式的服务器需验证方法为GET，否则返回400错误

## Code Examples
```c
/* Linux多线程Web服务器核心实现 */
void* request_handler(void* arg);
void send_data(FILE* fp, char* ct, char* file_name);
char* content_type(char* file);

int main(int argc, char *argv[])
{
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    pthread_t t_id;

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    /* ... bind, listen ... */

    while (1)
    {
        clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_size);
        printf("Connection Request : %s:%d\n",
            inet_ntoa(clnt_adr.sin_addr), ntohs(clnt_adr.sin_port));
        /* 为每个客户端请求创建线程 */
        pthread_create(&t_id, NULL, request_handler, &clnt_sock);
        pthread_detach(t_id);
    }
    close(serv_sock);
    return 0;
}

/* HTTP请求处理函数 */
void* request_handler(void *arg)
{
    int clnt_sock = *((int*)arg);
    char req_line[SMALL_BUF];
    FILE* clnt_read;
    FILE* clnt_write;
    char method[10];
    char ct[15];
    char file_name[30];

    /* 使用标准I/O分离读写流（第15-16章技术） */
    clnt_read = fdopen(clnt_sock, "r");
    clnt_write = fdopen(dup(clnt_sock), "w");

    /* 读取请求行 */
    fgets(req_line, SMALL_BUF, clnt_read);
    if (strstr(req_line, "HTTP/") == NULL) {
        send_error(clnt_write);
        fclose(clnt_read);
        fclose(clnt_write);
        return;
    }

    /* 解析请求方法和文件名 */
    strcpy(method, strtok(req_line, " /"));
    strcpy(file_name, strtok(NULL, " /"));
    strcpy(ct, content_type(file_name));

    /* 验证是否为GET请求 */
    if (strcmp(method, "GET") != 0) {
        send_error(clnt_write);
        fclose(clnt_read);
        fclose(clnt_write);
        return;
    }

    fclose(clnt_read);
    send_data(clnt_write, ct, file_name);  /* 发送响应 */
}

/* 发送HTTP响应数据 */
void send_data(FILE* fp, char* ct, char* file_name)
{
    char protocol[] = "HTTP/1.0 200 OK\r\n";
    char server[] = "Server:Linux Web Server\r\n";
    char cnt_len[] = "Content-length:2048\r\n";
    char cnt_type[SMALL_BUF];
    char buf[BUF_SIZE];
    FILE* send_file;

    sprintf(cnt_type, "Content-type:%s\r\n\r\n", ct);
    send_file = fopen(file_name, "r");
    if (send_file == NULL) {
        send_error(fp);
        return;
    }

    /* 发送响应头 */
    fputs(protocol, fp);
    fputs(server, fp);
    fputs(cnt_len, fp);
    fputs(cnt_type, fp);

    /* 发送请求的文件数据 */
    while (fgets(buf, BUF_SIZE, send_file) != NULL) {
        fputs(buf, fp);
        fflush(fp);
    }
    fflush(fp);
    fclose(fp);
}

/* 根据文件扩展名返回Content-type */
char* content_type(char* file)
{
    char extension[SMALL_BUF];
    char file_name[SMALL_BUF];
    strcpy(file_name, file);
    strtok(file_name, ".");
    strcpy(extension, strtok(NULL, "."));
    if (!strcmp(extension, "html") || !strcmp(extension, "htm"))
        return "text/html";
    else
        return "text/plain";
}
```
- **What it demonstrates**: 多线程HTTP服务器的完整实现——接受连接、创建线程、解析请求行、读取文件、构造HTTP响应并发送

## Reference Tables
| HTTP请求消息组成 | 内容 | 示例 |
|----------------|------|------|
| 请求行 | 方法 文件名 协议版本 | GET /index.html HTTP/1.1 |
| 消息头 | 浏览器信息、认证信息等 | User-Agent: Mozilla/5.0 |
| 消息体 | POST方式传输的数据 | （GET方式无此部分） |

| HTTP响应消息组成 | 内容 | 示例 |
|----------------|------|------|
| 状态行 | 协议版本 状态码 状态描述 | HTTP/1.1 200 OK |
| 头信息 | 服务器名、数据类型、长度 | Content-type: text/html |
| 消息体 | 请求的文件数据 | HTML文件内容 |

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | OK | 成功处理请求 |
| 404 | Not Found | 请求的文件不存在 |
| 400 | Bad Request | 请求方式错误 |

| Content-type值 | 文件类型 |
|----------------|---------|
| text/html | HTML文件(.html, .htm) |
| text/plain | 纯文本文件 |

## Key Takeaways
1. HTTP是无状态（Stateless）协议，服务器响应请求后立即断开连接，不维护客户端状态
2. 请求消息由请求行（方法+文件名+协议版本）、消息头和消息体组成，请求行仅1行
3. GET方法用于请求数据，POST用于传输数据；简单Web服务器只需响应GET请求
4. 响应消息由状态行（状态码）、头信息（Content-type/Content-length）和消息体（文件数据）组成
5. Web服务器采用多线程模型而非IOCP/epoll，因为HTTP连接太短暂无法发挥异步I/O优势
6. Linux版服务器可结合fdopen+dup使用标准I/O处理HTTP请求（第15-16章技术的综合应用）

## Connects To
- **Ch 15-16**: Linux版Web服务器使用fdopen+dup分离I/O流来解析HTTP请求，是标准I/O和流分离的综合应用
- **Ch 18-19**: HTTP服务器使用pthread_create(Linux)或_beginthreadex(Windows)为每个请求创建线程
- **Ch 17/23**: HTTP因Stateless特性不适合IOCP/epoll，需长连接场景（如游戏服务器）才能发挥优势
