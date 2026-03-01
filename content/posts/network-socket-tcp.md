+++
title = "Day 5：网络编程基础——Socket 原理与 TCP 实战"
date = 2026-03-01T15:00:00+08:00
draft = false
weight = 5
tags = ["C++", "网络编程", "Socket", "TCP", "多线程", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：这是第 5 天，正式进入网络编程。昨天理解了文件描述符（fd）和 inode，今天发现 Socket 不过是 fd 的另一种形态——Linux 里"一切皆文件"的设计哲学在这里体现得淋漓尽致。今天完成了单客户端 echo server 和多线程版本，并用 `nc` 工具验证了两个客户端同时通信。

---

## 1. Socket 的本质

Socket 就是一个**文件描述符**。

昨天用 `open()` 打开文件拿到一个 fd，用 `read()` / `write()` 读写，用 `close()` 关闭。Socket 完全一样，只是"文件"的另一端不是磁盘，而是网络上的另一台机器：

```
普通文件：  open()   → fd → read()/write() → close()
Socket：   socket() → fd → recv()/send()  → close()
```

操作系统内核负责把 `send()` 的数据打包成 TCP 报文发出去，把收到的报文解包后放进缓冲区等 `recv()`。上层代码只管读写 fd，不需要关心底层细节。

---

## 2. TCP vs UDP

| | TCP | UDP |
|---|---|---|
| 连接 | 需要三次握手 | 无连接，直接发 |
| 可靠性 | 保证送达、保证顺序 | 不保证，可能丢包乱序 |
| 速度 | 较慢（有确认机制）| 快 |
| 适用场景 | 文件传输、HTTP、SSH | 视频直播、DNS、游戏 |

客户端传输文件用 **TCP**——文件不能丢包，少一个字节文件就损坏了。直播用 **UDP**——丢几帧用户感知不明显，但为等重传导致的卡顿体验更差。

---

## 3. 三次握手与四次挥手

### 三次握手：建立连接

```
客户端                        服务端
  |  ──── SYN（我要连接你）──→  |
  |  ←─── SYN+ACK（好的收到）── |
  |  ──── ACK（我知道了）──────→ |
  |          连接建立            |
```

**为什么是三次而不是两次？**

两次握手的问题是：服务端回了 SYN+ACK，但如果这个 ACK 在网络里丢了，客户端以为连接没建立，服务端却以为连接成功，双方状态不一致。第三次 ACK 的目的是让服务端确认"客户端确实收到了我的响应"。

**如果第三次 ACK 丢了会怎样？**

```
客户端状态：ESTABLISHED         服务端状态：SYN_RCVD
（以为连接成功了）               （还在等 ACK，没收到）
```

服务端超时后会重发 SYN+ACK，客户端收到后再次发 ACK，连接最终仍可建立。极端情况下服务端资源被大量半连接占满，这也是 **SYN Flood 攻击**的原理。

### 四次挥手：断开连接

```
主动方                         被动方
  |  ──── FIN（我说完了）──────→ |
  |  ←─── ACK（收到）─────────── |
  |  ←─── FIN（我也说完了）────── |
  |  ──── ACK（收到，再见）──────→|
```

断开比建立多一次，是因为 TCP 是**全双工**的——两个方向的数据流独立关闭，一方说完了另一方可能还有数据要发，不能合并成一次。

---

## 4. TCP Server 的四步流程

```
socket() → bind() → listen() → accept()
创建fd     绑定地址   开始监听    接受连接
```

用电话类比：
- `socket()`：买了一部电话
- `bind()`：给电话装上固定号码
- `listen()`：开机，等待来电
- `accept()`：接听一通来电，拿到这通电话的专属 fd

### server_fd 和 client_fd 的区别

这是今天最重要的概念之一。**两个 fd 都存在于服务端程序里**，客户端程序里一个都没有：

```
服务端程序内部：
┌─────────────────────────────────┐
│  server_fd                      │  ← 专门负责等新连接
│                                 │
│  client_fd_1  ←→ 客户端A        │  ← accept() 返回的
│  client_fd_2  ←→ 客户端B        │  ← accept() 返回的
└─────────────────────────────────┘
```

- `server_fd`：前台总机，只做 `accept()`，永远不直接收发数据
- `client_fd`：转接后的专线，每个客户端连接对应一个，用来 `recv()` / `send()`

---

## 5. 代码实战

### 单客户端版本

```cpp
#include <iostream>
#include <string>
#include <cstring>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);

    // 避免端口复用问题（见踩坑部分）
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);        // 注意字节序转换
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(server_fd, (sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 10);
    std::cout << "服务端启动，监听 8888 端口...\n";

    while (true) {
        sockaddr_in client_addr{};
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(server_fd, (sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) continue;

        std::cout << "新客户端连接: " << inet_ntoa(client_addr.sin_addr) << "\n";

        char buf[1024];
        while (true) {
            ssize_t n = recv(client_fd, buf, sizeof(buf) - 1, 0);
            if (n <= 0) break;
            buf[n] = '\0';
            std::cout << "收到: " << buf;
            send(client_fd, buf, n, 0);
        }
        close(client_fd);
    }
    close(server_fd);
}
```

### 多线程版本（每个连接一个线程）

```cpp
#include <iostream>
#include <string>
#include <thread>
#include <cstring>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

void handle_client(int client_fd, std::string client_ip) {
    std::cout << "[" << client_ip << "] 已连接\n";
    char buf[1024];
    while (true) {
        ssize_t n = recv(client_fd, buf, sizeof(buf) - 1, 0);
        if (n <= 0) break;
        buf[n] = '\0';
        std::cout << "[" << client_ip << "] 收到: " << buf;
        send(client_fd, buf, n, 0);
    }
    std::cout << "[" << client_ip << "] 已断开\n";
    close(client_fd);
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
    std::cout << "服务端启动，监听 8888 端口...\n";

    while (true) {
        sockaddr_in client_addr{};
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(server_fd, (sockaddr*)&client_addr, &client_len);
        if (client_fd < 0) continue;

        std::string ip = inet_ntoa(client_addr.sin_addr);
        // server_fd 永远在主线程，client_fd 扔给子线程
        std::thread(handle_client, client_fd, ip).detach();
    }
    close(server_fd);
}
```

**编译命令：**
```bash
g++ -g -pthread -o server server.cpp
./server

# 另开终端用 nc 测试
nc 127.0.0.1 8888
```

---

## 6. 字节序：htons 是干什么的

网络协议统一用**大端（Big Endian）**，而 x86/ARM 通常用**小端（Little Endian）**。往 `sockaddr` 里写数据必须先转换：

```cpp
htons(8888)   // 端口号：本机 → 网络字节序（16位）
ntohs(port)   // 端口号：网络 → 本机字节序
htonl(ip)     // IP地址：本机 → 网络字节序（32位）
ntohl(addr)   // IP地址：网络 → 本机字节序
```

**为什么 `INADDR_ANY` 不需要转换？**

`INADDR_ANY` 的值是 `0x00000000`，每个字节都是 0，大端小端翻转后结果相同，所以碰巧不需要转换。不是因为"它已经定义好了"，而是因为 **0 的字节序翻转结果仍是 0**。

口诀：**往 `sockaddr` 里写的数据必须转成网络字节序，读出来之后要转回本机字节序。**

---

## 7. 今日踩坑复盘

**坑一：编译产物名和运行名不一致**

```bash
g++ -g -o note server.cpp  # 产物叫 note
./server                    # ❌ 找不到
./note                      # ✅ 正确
```

**坑二：bind 失败——端口被占用**

程序崩溃后端口不会立刻释放，重新运行时 `bind()` 报 "Address already in use"。解决方法是提前加一行：

```cpp
int opt = 1;
setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

**坑三：recv() 不保证一次读完完整消息**

TCP 是流协议，没有消息边界。客户端发 `"hello\n"`，`recv()` 可能分两次返回 `"hel"` 和 `"lo\n"`。正确做法是把收到的数据拼接起来，找到分隔符才算完整一条消息：

```cpp
std::string buffer;
while (true) {
    char chunk[256];
    ssize_t n = recv(client_fd, chunk, sizeof(chunk), 0);
    if (n <= 0) break;
    buffer.append(chunk, n);

    size_t pos;
    while ((pos = buffer.find('\n')) != std::string::npos) {
        std::string line = buffer.substr(0, pos);
        buffer.erase(0, pos + 1);
        process(line);  // 处理完整的一行
    }
}
```

**坑四：服务端崩溃时客户端 send() 不会立刻报错**

第一次 `send()` 数据进了内核缓冲区通常会成功，第二次才会收到 `SIGPIPE` 导致进程退出。解决方法：

```cpp
#include <signal.h>
signal(SIGPIPE, SIG_IGN);  // 忽略 SIGPIPE，让 send() 返回 -1
```

---

## 💡 核心总结

```
Socket = 文件描述符 + 网络另一端
server_fd = 只接新连接，永远不收发数据
client_fd = 每个客户端一个，专门用来通信
recv/send 不保证完整 → 需要自己处理粘包
字节序转换 → 写入 sockaddr 时必做
SO_REUSEADDR → 开发阶段必加
```

> **记住**：`server_fd` 和 `client_fd` 都在服务端，前者是总机，后者是专线。`INADDR_ANY` 不需要转字节序不是因为它"已经定义好了"，而是因为它的值是 0，翻转字节序后结果不变。

---