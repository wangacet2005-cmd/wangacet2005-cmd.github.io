+++
title = "Day 13：项目开发——多线程传输重构、断点续传与多文件队列"
date = 2026-03-13T21:00:00+08:00
draft = false
weight = 13
tags = ["C++", "Qt", "多线程", "断点续传", "MD5", "moveToThread", "项目实战", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：Day 12 实现了基本的文件传输，但传输逻辑跑在主线程，大文件会卡 UI。今天做三件事：把传输逻辑移到子线程、加上断点续传、实现多文件队列传输。断点续传是面试亮点功能，能讲清楚实现原理会很加分。

---

## 1. 架构重构：从主线程到 Worker 子线程

### Day 12 的问题

Day 12 的 TransferServer 和 TransferClient 直接在主线程运行。虽然用了 QTimer::singleShot(0) 让出事件循环，但文件读写、MD5 计算这些 CPU 密集操作仍然在主线程执行，大文件传输时 UI 会有轻微卡顿。

### 重构方案

把收发逻辑拆成两个 Worker 类：SendWorker 负责发送，RecvWorker 负责接收，各自通过 moveToThread 跑在独立子线程里。MainWindow 只管发信号和接信号，完全不参与传输逻辑。

```
MainWindow（主线程）
    │
    │── emit requestSend() ──→ SendWorker（发送线程）
    │← progress/finished ────│
    │
    │── emit requestListen() ─→ RecvWorker（接收线程）
    │← progress/finished ────│
```

### Worker 不能有 parent

这是 moveToThread 的老规矩：Worker 对象不传 parent，生命周期通过 `QThread::finished → deleteLater` 管理。析构函数里必须 `quit() + wait()` 确保线程安全退出：

```cpp
MainWindow::~MainWindow() {
    m_sendThread->quit();
    m_sendThread->wait();
    m_recvThread->quit();
    m_recvThread->wait();
}
```

### 重构后的好处

传输过程中拖动窗口、点击按钮、选择文件都完全流畅。网络层和 UI 层通过信号槽通信，没有任何共享变量，线程安全问题从设计上消除。

---

## 2. MD5 文件校验

### 双端同步计算

发送端每读一块数据，先喂给 QCryptographicHash 再发出去：

```cpp
QByteArray chunk = m_file->read(CHUNK_SIZE);
m_hash.addData(chunk);          // 喂给 MD5
sendPacket(FileData, chunk);     // 发出去
```

接收端每收一块数据，也喂给 QCryptographicHash 再写入文件：

```cpp
m_file->write(payload);
m_hash.addData(payload);         // 喂给 MD5
```

文件传完后，发送端把自己的 MD5 哈希值放在 FileComplete 的 payload 里发过去，接收端拿本地计算结果比对。一致说明传输无损，不一致说明中间丢了或改了数据。

### 为什么不用传输后再算

如果传完再对整个文件算 MD5，需要重新读一遍文件，大文件很慢。边传边算只增加极少的计算开销（每块 64KB 的 MD5 update 操作几乎不耗时），传输完成时 MD5 就已经算好了。

---

## 3. 断点续传

### 核心原理

接收端用 `.part` 临时文件保存未完成的数据。下次收到同一文件的 FileRequest 时，检测到 `.part` 文件存在且大小小于目标文件大小，回复 ResumeRequest 告知已接收字节数。发送端收到后用 `file.seek()` 跳到对应位置继续发送。传输完成后 `.part` 重命名为正式文件。

```
第一次传输（中途断开）：
  发送端 → FileRequest → 接收端创建 file.zip.part
  发送端 → FileData × N → 接收端写入 .part
  断开 → .part 保留，记录了已接收的字节数

第二次传输（续传）：
  发送端 → FileRequest → 接收端发现 .part 存在
  接收端 → ResumeRequest(已收 X 字节) → 发送端
  发送端 → file.seek(X)，从 X 处继续发 FileData
  完成 → .part 重命名为 file.zip
```

### 接收端的关键逻辑

```cpp
QFileInfo partInfo(partPath);
if (partInfo.exists() && partInfo.size() > 0
    && partInfo.size() < info.fileSize) {
    // 有未完成文件，重新计算已有部分的 MD5
    m_receivedBytes = partInfo.size();
    QFile existingFile(partPath);
    existingFile.open(QIODevice::ReadOnly);
    while (!existingFile.atEnd()) {
        m_hash.addData(existingFile.read(64 * 1024));
    }
    existingFile.close();

    // 发送 ResumeRequest
    // 以追加模式打开 .part 文件
    m_file = new QFile(partPath, this);
    m_file->open(QIODevice::Append);
}
```

两个细节容易忽略：一是续传时必须重新读取 `.part` 文件计算已有部分的 MD5，否则最终的 MD5 只覆盖续传部分，校验必然失败。二是文件必须以 Append 模式打开，WriteOnly 会清空已有数据。

### 发送端的关键逻辑

```cpp
case Protocol::MessageType::ResumeRequest: {
    Protocol::ResumeInfo resume =
        Protocol::ResumeInfo::deserialize(payload);
    // 跳过已传输部分
    m_file->seek(resume.alreadyReceived);
    m_sentBytes = resume.alreadyReceived;
    // 重新计算已跳过部分的 MD5
    // ... 读取前 X 字节喂给 hash
    QTimer::singleShot(0, this, &SendWorker::sendNextChunk);
    break;
}
```

发送端也要重新计算已跳过部分的 MD5，否则发送端的最终哈希值只覆盖后半段，和接收端的全文件哈希对不上。

### 面试怎么讲

一句话版本：接收端用临时文件记录进度，下次传输时告知发送端已收字节数，发送端 seek 到对应位置继续发，两端 MD5 覆盖完整文件保证一致性。

追问"如果 .part 文件损坏了怎么办"：可以在 ResumeRequest 里附带已接收部分的 MD5，发送端比对后决定续传还是重传。当前实现没做这一步，但知道这个优化方向就够了。

---

## 4. 多文件队列传输

### 设计思路

MainWindow 维护一个 `QStringList m_sendQueue` 作为发送队列，一个 `m_currentSendIndex` 记录当前发到第几个。每个文件传输完成后，`onTransferFinished` 自动把 index +1 并调用 `sendNextFile` 发下一个。全部发完后恢复按钮状态。

```cpp
void MainWindow::sendNextFile() {
    if (m_currentSendIndex >= m_sendQueue.size()) {
        // 全部发完
        return;
    }
    QString path = m_sendQueue[m_currentSendIndex];
    emit requestSend(m_targetHost, 9999, path);
}

void MainWindow::onTransferFinished(...) {
    // 更新当前文件状态
    m_currentSendIndex++;
    QTimer::singleShot(500, this, &MainWindow::sendNextFile);
}
```

### 为什么串行不并行

多文件并行传输会引入很多复杂性：多个 Socket 竞争带宽、接收端要同时写多个文件、进度跟踪变得混乱。串行传输简单可靠，带宽利用率也不差（一个文件传完立刻开始下一个）。面试如果被问"为什么不并行"，就说这个理由。

### 500ms 延迟的作用

`QTimer::singleShot(500, ...)` 在两个文件之间加了 500 毫秒的间隔，给接收端时间完成文件关闭、重命名、状态重置。没有这个延迟，下一个 FileRequest 可能在接收端还没处理完上一个文件时就到了。

---

## 5. 协议文件的组织方式

### 为什么 .cpp 只有一行

transferprotocol.h 里的序列化函数都写在结构体内部，编译器自动当作 inline。transferprotocol.cpp 只需要 `#include "core/transferprotocol.h"` 这一行，保证 CMake 有编译单元就行。

这种"头文件搞定一切 + .cpp 只有 include"的写法对于轻量级的数据结构类是完全合理的。如果结构体的方法变复杂了（比如加了加密、压缩），再把实现拆到 .cpp 里也不迟。

---

## 6. Day 12 → Day 13 的知识衔接

| Day 12 | Day 13 |
|---|---|
| TransferServer/Client 在主线程 | SendWorker/RecvWorker 通过 moveToThread 在子线程 |
| 单文件传输 | 多文件队列串行传输 |
| 传完对比 MD5 | 边传边算 MD5，续传时重算已有部分 |
| 传输中断 = 数据丢失 | .part 临时文件 + ResumeRequest 实现断点续传 |
| MainWindow 直接调用网络对象方法 | MainWindow 只 emit 信号，Worker 在子线程执行 |

---

## 核心总结

架构重构的核心是把传输逻辑从主线程移到 Worker 子线程，通过信号槽通信消除共享变量，线程安全从设计上保证。MD5 校验边传边算，不需要传完再算一遍。断点续传靠三个机制配合：.part 临时文件保存进度、ResumeRequest 协议协商续传位置、seek 跳过已传部分。续传时两端都必须重算已有部分的 MD5，否则最终校验必然失败。多文件串行传输简单可靠，两个文件之间加延迟给接收端处理时间。

> **记住**：面试问断点续传，核心就三句话——接收端用临时文件记录已收字节数，通过 ResumeRequest 告知发送端，发送端 seek 到对应位置继续发。追问 MD5 怎么处理，答"两端都重算已有部分的哈希，确保最终校验覆盖完整文件"。

明天（Day 14）进入多线程传输和 UI 集成的优化，加上进度条联动和传输速度显示。

---
