+++
title = "Day 6：高性能网络 I/O——从阻塞到 epoll"
date = 2026-03-06T21:00:00+08:00
draft = false
weight = 6
tags = ["C++", "网络编程", "epoll", "非阻塞IO", "高并发", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：这是第 6 天。昨天的多线程 Server 每来一个连接就创建一个线程，1 万个连接就要 1 万个线程，内存和切换开销直接把程序拖垮。今天解决这个问题，理解 Linux 高性能网络 I/O 的核心机制：非阻塞 I/O 和 epoll。最终用单线程实现了同时处理多个客户端连接的 echo server。

---

## 1. 昨天方案的根本缺陷

先量化一下问题有多严重：

```
一个线程默认栈大小：8MB
1000 个连接 = 1000 个线程 = 8GB 内存
仅仅是栈空间就已经不可接受了
```

更深层的问题是：大多数时候这些线程都在**阻塞等待**，等客户端发数据过来。它们什么都不干，却占着内存和系统资源。

```cpp
// 昨天的代码：recv() 会一直卡在这里，直到有数据来
ssize_t n = recv(client_fd, buf, sizeof(buf), 0);
//              ↑ 阻塞：线程挂起，等待内核通知
```

这就是**阻塞 I/O** 的本质——线程把自己交给内核，内核说"有数据了再叫你"，线程在此期间什么也干不了。

---

## 2. 非阻塞 I/O

把 fd 设置为非阻塞模式之后，`recv()` 不再等待，立刻返回：

```cpp
#include <fcntl.h>

void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// 非阻塞模式下的 recv
ssize_t n = recv(client_fd, buf, sizeof(buf), 0);
if (n < 0) {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 不是出错，只是暂时没有数据，待会儿再试
    } else {
        // 真正的错误
    }
}
```

但随之而来的问题是：**怎么知道哪个 fd 有数据？** 如果用循环轮询所有 fd，CPU 会被白白消耗在无效检查上，这叫**忙等待（Busy Waiting）**，不可接受。

真正需要的是：让内核盯着所有 fd，有事件发生时主动通知我。这就是 **I/O 多路复用**。

---

## 3. select vs epoll

### select（了解即可）

```cpp
fd_set read_fds;
FD_ZERO(&read_fds);
FD_SET(fd1, &read_fds);
select(max_fd + 1, &read_fds, nullptr, nullptr, nullptr);

// 返回后要遍历所有 fd 才能知道谁就绪
for (int fd = 0; fd <= max_fd; fd++) {
    if (FD_ISSET(fd, &read_fds)) { /* 处理 */ }
}
```

**缺陷：**
- fd 数量上限 1024
- 每次调用都要把所有 fd 从用户空间拷贝到内核空间，O(n) 开销
- 返回后仍需遍历全部 fd 找出谁就绪，连接越多越慢

### epoll（Linux 现代方案）

epoll 和 select 最根本的区别不是 fd 上限，而是**复杂度**：

```
select：每次调用都是 O(n)，重新拷贝和遍历所有 fd
epoll：用持久的内核事件表，只返回就绪的 fd，是 O(1) 的
       连接数越多，优势越明显
```

三个核心函数：

```cpp
// 1. 创建 epoll 实例
int epfd = epoll_create1(0);

// 2. 注册/修改/删除监听的 fd
struct epoll_event ev;
ev.events = EPOLLIN;        // 监听可读事件
ev.data.fd = client_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);   // 添加
epoll_ctl(epfd, EPOLL_CTL_DEL, client_fd, nullptr); // 删除

// 3. 等待事件，直接返回就绪的 fd 列表
struct epoll_event events[64];
int n = epoll_wait(epfd, events, 64, -1);  // -1 永久等待

for (int i = 0; i < n; i++) {
    int fd = events[i].data.fd;
    // 只处理真正就绪的 fd
}
```

---

## 4. 两种触发模式

### 水平触发（LT，默认）

只要 fd 上还有数据没读完，每次 `epoll_wait` 都会通知。好写不容易漏数据，是大多数场景的推荐选择。

### 边缘触发（ET）

```cpp
ev.events = EPOLLIN | EPOLLET;  // 开启 ET 模式
```

只在状态**发生变化的那一刻**通知一次，之后不再通知。性能更好，但**必须一次性把数据全部读完**，否则剩余数据永远躺在缓冲区里，再也不会触发通知。

```cpp
// ET 模式下必须循环读到 EAGAIN
while (true) {
    ssize_t n = recv(fd, buf, sizeof(buf), 0);
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) break;  // 读完了
        break;  // 真正的错误
    }
    if (n == 0) break;  // 对方断开
    // 处理数据...
}
```

---

## 5. 完整的 epoll Server

```cpp
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>

void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(server_fd, (sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 10);

    set_nonblocking(server_fd);  // server_fd 也要设为非阻塞（见踩坑部分）

    int epfd = epoll_create1(0);
    epoll_event ev{};
    ev.events = EPOLLIN;
    ev.data.fd = server_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

    std::cout << "epoll server 启动，监听 8888...\n";

    epoll_event events[64];

    while (true) {
        int n = epoll_wait(epfd, events, 64, -1);

        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == server_fd) {
                // 有新连接进来
                sockaddr_in client_addr{};
                socklen_t len = sizeof(client_addr);
                int client_fd = accept(server_fd, (sockaddr*)&client_addr, &len);
                if (client_fd < 0) continue;  // 非阻塞：失败直接跳过

                set_nonblocking(client_fd);

                epoll_event cev{};
                cev.events = EPOLLIN;
                cev.data.fd = client_fd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &cev);

                std::cout << "新连接: " << inet_ntoa(client_addr.sin_addr) << "\n";

            } else {
                // 普通客户端有数据可读
                char buf[1024];
                ssize_t bytes = recv(fd, buf, sizeof(buf) - 1, 0);

                if (bytes <= 0) {
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, nullptr);
                    close(fd);
                    std::cout << "客户端断开 fd=" << fd << "\n";
                } else {
                    buf[bytes] = '\0';
                    std::cout << "收到 fd=" << fd << ": " << buf;
                    send(fd, buf, bytes, 0);
                }
            }
        }
    }

    close(epfd);
    close(server_fd);
    return 0;
}
```

**编译运行：**
```bash
g++ -g -o epoll_server epoll_server.cpp
./epoll_server
```

---

## 6. 与昨天多线程方案的本质区别

```
昨天（多线程）：
1 个连接 = 1 个线程
10000 个连接 = 10000 个线程
大部分线程在阻塞等待，浪费资源

今天（epoll）：
1 个线程管理所有连接
只有真正有数据的 fd 才触发处理
线程几乎不阻塞，CPU 利用率高
```

这就是 **Reactor 模式**的核心思想——用事件驱动代替线程驱动。Nginx 就是用这个模型支撑百万并发连接的。

---

## 7. 今日踩坑复盘

**为什么 server_fd 也要设成非阻塞？**

这是今天最容易忽略的细节。`server_fd` 设成非阻塞是为了防止一个竞态条件：

```
1. epoll_wait 返回，告诉你 server_fd 可读（有新连接）
2. 但在你调用 accept() 之前，客户端突然重置了连接
3. 此时连接队列变空了

如果 server_fd 是阻塞的：
accept() 会一直卡住，等下一个连接，整个事件循环冻结
其他 fd 上的事件全部积压，没人处理

如果 server_fd 是非阻塞的：
accept() 立刻返回 -1，errno = EAGAIN
跳过这次，继续处理其他事件，事件循环正常运转
```

**ET 模式只读了一半的后果**

剩余数据永远躺在缓冲区里，不会再触发通知，也不会消失。直到对方再发新数据触发边缘变化，两段数据混在一起，逻辑彻底乱掉。ET 模式下收到通知必须循环读到 `EAGAIN`。

---

## 💡 核心总结

```
select  → O(n)，每次重新拷贝遍历所有 fd，上限 1024
epoll   → O(1)，内核持久维护事件表，直接返回就绪列表
LT 模式 → 有数据就通知，好写不易漏，推荐默认使用
ET 模式 → 只通知一次，必须一次读完，性能更好但容易出 bug
server_fd 非阻塞 → 防止 accept() 卡死事件循环
```

> **记住**：epoll 和 select 最根本的区别是 O(1) vs O(n)，fd 上限只是表象。ET 模式下收到通知后必须循环读到 `EAGAIN`，否则剩余数据永远丢失。

明天（Day 7）在 epoll 的基础上完成题目七的 TCP Echo Server，加上完整的错误处理、quit 命令支持和多客户端并发，把这三天的网络编程知识融合成一个完整的项目。

---