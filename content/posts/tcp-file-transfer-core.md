+++
title = "Day 12：项目开发——主界面搭建与 TCP 文件传输核心实现"
date = 2026-03-12T21:00:00+08:00
draft = false
weight = 12
tags = ["C++", "Qt", "TCP", "文件传输", "协议设计", "粘包处理", "项目实战", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：项目框架搭好了，今天开始填充真正的功能。目标是让程序能通过 TCP 把一个文件从发送端传到接收端，进度条实时更新，接收到的文件内容正确。这是整个项目最核心的一天——传输协议的实现、粘包处理的落地、以及收发两端的状态管理。

---

## 1. 整体传输流程

一次完整的文件传输分六步：

```
发送端                              接收端
  │                                    │
  │── FileRequest(文件名+大小) ───────→│ 创建文件准备写入
  │←── FileAccept ────────────────────│
  │── FileData(数据块1) ──────────────→│ 写入文件
  │── FileData(数据块2) ──────────────→│ 写入文件
  │── ... ────────────────────────────→│ ...
  │── FileComplete ───────────────────→│ 关闭文件，传输结束
```

发送端先发 FileRequest 告知文件名和大小，接收端同意后回 FileAccept，然后发送端开始分块发数据，每块 64KB，全部发完后发 FileComplete 通知结束。任何一方都可以发 FileCancel 中止传输。

---

## 2. 粘包处理的实际落地

### 状态机设计

Day 7 讲了粘包的原理，今天把它落地到代码里。用一个布尔变量 `m_waitingHeader` 实现两状态的状态机：

```
等包头状态 → 缓冲区够12字节 → 解析包头，拿到payloadSize → 切换到等数据体状态
等数据体状态 → 缓冲区够payloadSize字节 → 取出数据，分发处理 → 切换回等包头状态
```

核心代码：

```cpp
void TransferServer::processPacket() {
    while (true) {
        if (m_waitingHeader) {
            if (m_buffer.size() < Protocol::PacketHeader::SIZE)
                return;  // 数据不够，等下次 readyRead
            // 解析包头...
            m_waitingHeader = false;
        }
        if (m_buffer.size() < static_cast<int>(m_currentHeader.payloadSize))
            return;  // 数据不够，等下次 readyRead
        // 取出 payload，分发处理...
        m_waitingHeader = true;
    }
}
```

关键点：外层是 `while(true)` 循环，因为一次 readyRead 可能带来多个完整的包，必须循环处理直到缓冲区里的数据不够一个完整包为止。

### 面试怎么讲

粘包本质是 TCP 是流协议没有消息边界。解决方案是固定长度包头 + 变长数据体，接收端维护一个缓冲区，每次 readyRead 把数据追加进去，然后用状态机尝试解析。状态机只有两个状态：等包头和等数据体。

---

## 3. 发送端的设计

### QTimer::singleShot 避免阻塞

发送大文件时不能用 for 循环一口气读完发完——那样会阻塞事件循环，UI 卡死，进度条不更新。解决方案是每发一块就用 `QTimer::singleShot(0, ...)` 把"发下一块"的操作扔回事件循环：

```cpp
void TransferClient::sendNextChunk() {
    QByteArray chunk = m_file->read(CHUNK_SIZE);  // 读 64KB
    if (chunk.isEmpty()) {
        // 文件读完，发送 FileComplete
        sendPacket(Protocol::MessageType::FileComplete);
        return;
    }
    sendPacket(Protocol::MessageType::FileData, chunk);
    m_sentBytes += chunk.size();
    emit transferProgress(m_sentBytes * 100 / m_fileSize);

    // 让出事件循环，UI 有机会更新，然后继续发下一块
    QTimer::singleShot(0, this, &TransferClient::sendNextChunk);
}
```

`QTimer::singleShot(0, ...)` 的延时是 0 毫秒，意思是"尽快执行，但先让事件循环处理完当前排队的事件"。这样每发一块数据，UI 都有机会刷新进度条。

### 为什么不用子线程

当前的发送逻辑跑在主线程就够了，因为文件读取和 Socket 写入都很快，真正的耗时在网络传输，而 Qt 的 Socket 是异步的。Day 14 会把传输逻辑移到子线程，处理更复杂的场景（多文件并行、大文件不阻塞文件选择等）。

---

## 4. 接收端的设计

### 收到 FileRequest 后的处理

接收端收到文件请求后，创建本地文件准备写入。如果文件创建失败（磁盘满、权限不足等），回复 FileReject：

```cpp
void TransferServer::handleFileRequest(const QByteArray &payload) {
    Protocol::FileInfo info = Protocol::FileInfo::deserialize(payload);
    m_file = new QFile(m_savePath + "/" + info.fileName, this);
    if (!m_file->open(QIODevice::WriteOnly)) {
        // 发送 FileReject
        sendPacket(Protocol::MessageType::FileReject);
        return;
    }
    // 发送 FileAccept
    sendPacket(Protocol::MessageType::FileAccept);
}
```

### 客户端断开时的清理

客户端可能在传输中途断开（网络异常、用户关闭程序），接收端必须清理状态：关闭文件、清空缓冲区、重置状态机。忘了清理缓冲区会导致下一个客户端连上来时收到脏数据——和 Day 7 的 `buffers.erase(fd)` 是完全一样的问题。

---

## 5. 主窗口集成

### 信号槽连接图

```
TransferServer                MainWindow              TransferClient
     │                            │                         │
     │── logMessage ─────────────→│ appendLog()             │
     │── transferProgress ───────→│ onUpdateProgress()      │
     │── transferComplete ───────→│ 更新状态栏               │
     │                            │                         │
     │                            │←── logMessage ──────────│
     │                            │←── transferProgress ────│
     │                            │←── connected ───────────│
     │                            │←── transferComplete ────│
```

Server 和 Client 都不知道 UI 长什么样，它们只管 emit 信号。MainWindow 通过 connect 把这些信号接到自己的槽函数上，更新进度条、状态栏、日志区。这就是信号槽解耦的好处——网络层和 UI 层完全独立。

### 连接服务器时弹出输入框

用 `QInputDialog::getText` 让用户输入服务器 IP，比硬编码 `127.0.0.1` 灵活：

```cpp
QString host = QInputDialog::getText(this, "连接服务器",
    "输入服务器IP地址:", QLineEdit::Normal, "127.0.0.1", &ok);
```

默认值填 `127.0.0.1` 方便本地测试，实际使用时用户可以改成局域网内其他机器的 IP。

---

## 6. 踩过的坑

### QToolBar 和 QStatusBar 的 incomplete type 错误

在 `mainwindow.cpp` 里调用 `toolbar->addWidget()` 和 `statusBar()->addWidget()` 时报错 "Member access into incomplete type 'QToolBar'" 和 "使用了未定义类型 QStatusBar"。

原因：`QMainWindow` 的头文件只对 QToolBar 和 QStatusBar 做了前向声明（forward declaration），没有包含它们的完整定义。前向声明告诉编译器"这个类存在"，但不告诉它类有哪些方法，所以调用方法时就报 incomplete type。

解决方案：在 `mainwindow.cpp` 顶部显式 include：

```cpp
#include <QToolBar>
#include <QStatusBar>
```

规则：如果你只用到指针声明（如 `QToolBar *toolbar`），前向声明就够了；但如果你要调用它的方法（如 `toolbar->addWidget()`），必须 include 完整的头文件。

### 为什么 Qt Creator 之前没报这个错

Day 8-9 的 demo 项目用的是 QPushButton、QLabel 这些控件，它们的头文件被 QWidget 间接包含了。QToolBar 和 QStatusBar 是 QMainWindow 特有的，QMainWindow 为了减少编译依赖只做了前向声明，所以必须手动 include。这种"A 包含了 B 的前向声明但没包含完整定义"的情况在 Qt 里很常见。

---

## Day 11 → Day 12 的知识衔接

| Day 11 概念 | Day 12 落地 |
|---|---|
| 12 字节固定包头协议设计 | processPacket 状态机实现粘包处理 |
| 项目目录结构 src/network/ | TransferServer 和 TransferClient 完整实现 |
| UserRole 存文件路径 | onSendFiles 从 Model 取路径发送文件 |
| 信号槽解耦 UI 和逻辑 | Server/Client 只 emit 信号，MainWindow 负责更新 UI |
| Day 7 的 buffers.erase(fd) | onClientDisconnected 清空缓冲区和文件句柄 |

---

## 核心总结

TCP 文件传输的核心是三件事：协议设计解决粘包、状态机解析数据流、信号槽解耦网络层和 UI 层。发送端用 QTimer::singleShot(0) 在每块数据之间让出事件循环，保证 UI 响应。接收端用两状态的状态机（等包头/等数据体）处理粘包，外层 while 循环处理一次 readyRead 携带多个包的情况。客户端断开时必须清理所有状态——文件句柄、缓冲区、状态机，否则下个客户端会收到脏数据。前向声明只够声明指针，调用方法必须 include 完整头文件。

> **记住**：面试问"你项目里是怎么处理粘包的"，就讲固定包头 + 状态机 + while 循环。面试官追问"为什么要 while 循环"，答"一次 readyRead 可能带来多个完整的包，不循环就会丢包"。这个回答比背概念强得多，因为你是真的写过。

明天（Day 13）把传输逻辑移到子线程，加上 MD5 文件校验，确保大文件传输时 UI 完全不卡且数据完整性可验证。

---
