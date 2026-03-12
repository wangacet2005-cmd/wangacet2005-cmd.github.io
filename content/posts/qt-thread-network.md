+++
title = "Day 10：Qt 多线程与网络模块——QThread、QTcpSocket 与跨线程信号槽"
date = 2026-03-10T21:00:00+08:00
draft = false
weight = 10
tags = ["C++", "Qt", "QThread", "QTcpSocket", "多线程", "网络编程", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：Day 5–7 用原生 API 写了多线程和 Socket，Day 8–9 学了 Qt 的信号槽和控件。今天把两边打通——用 Qt 的方式重新实现多线程和网络编程。核心是 moveToThread 的正确用法和跨线程信号槽的底层原理，这两个知识点面试几乎必问。

---

## 1. QThread 的两种用法

### moveToThread（推荐方式）

思路是"把工作搬到线程里"：Worker 对象负责业务逻辑，QThread 只负责提供线程环境和事件循环。

四个步骤缺一不可：

```cpp
// 第1步：创建线程和 Worker（Worker 不能有 parent）
m_workerThread = new QThread(this);
m_worker = new FileWorker();  // 不传 parent

// 第2步：把 Worker 移到子线程
m_worker->moveToThread(m_workerThread);

// 第3步：连接信号槽
connect(this, &ThreadDemo::startWork, m_worker, &FileWorker::doWork);
connect(m_worker, &FileWorker::progress, m_progressBar, &QProgressBar::setValue);
connect(m_workerThread, &QThread::finished, m_worker, &QObject::deleteLater);

// 第4步：启动线程
m_workerThread->start();
```

Worker 不能有 parent 是因为 moveToThread 要求对象的线程亲和性可以改变，有 parent 的对象被 parent 的线程"拥有"，无法移动。

### 继承 QThread（了解即可）

```cpp
class LegacyThread : public QThread {
protected:
    void run() override {
        // 只有这个函数运行在新线程里
        // 其他成员函数（包括槽函数）仍然在主线程执行
    }
};
```

这种方式的陷阱在于：只有 run() 在子线程，类的其他槽函数都在创建该对象的线程（通常是主线程）。很多人以为继承了 QThread 就全在子线程，这是错的。

### 面试对比表

|  | moveToThread | 继承 QThread |
|---|---|---|
| Worker 的槽在哪个线程执行 | 子线程 | 主线程（除了 run()） |
| 能用信号触发工作吗 | 能，天然支持 | 不能，只有 run() 在子线程 |
| 线程有事件循环吗 | 有 | 默认没有 |
| 能处理多个任务吗 | 能 | 不能，run() 跑完就结束 |

一句话：moveToThread 是"把工作搬到线程里"，继承 QThread 是"把线程当函数跑"。

---

## 2. 跨线程信号槽的底层原理

这是今天最重要的知识点。

### 自动连接机制

Qt 的 connect 默认是 `Qt::AutoConnection`，它在运行时判断信号发射方和槽接收方是不是同一个线程：

- **同线程**：直接调用槽函数（DirectConnection），和普通函数调用一样
- **跨线程**：把信号参数打包成 QEvent，投递到目标线程的事件队列（QueuedConnection）

```
子线程 emit progress(50)
    → Qt 检测到跨线程
    → 包装成 QMetaCallEvent，放入主线程事件队列
    → 主线程事件循环取出
    → 在主线程执行 progressBar->setValue(50)
```

这就是为什么子线程 emit 信号能安全更新 UI——槽函数不是在子线程执行的，而是在主线程排队执行的。

### 绝对禁止的操作

在子线程里直接调用 UI 控件的方法，必崩溃：

```cpp
// ❌ 在 Worker 的 doWork 里直接操作 UI
m_label->setText("完成");     // 崩溃
m_progressBar->setValue(100); // 崩溃

// ✅ 只能通过信号间接更新
emit progress(100);           // 信号自动投递到主线程
emit finished("完成");
```

Qt 的 GUI 子系统不是线程安全的，所有 QWidget 的操作必须在主线程执行，没有例外。

---

## 3. 线程的正确销毁

析构函数里必须按顺序清理：

```cpp
ThreadDemo::~ThreadDemo() {
    m_worker->stop();          // 通知 Worker 停止工作
    m_workerThread->quit();    // 请求线程退出事件循环
    m_workerThread->wait();    // 阻塞等待线程真正结束
}
```

少了 quit() 线程不会退出，少了 wait() 主线程可能在子线程还在运行时就销毁了对象，导致悬空指针崩溃。`deleteLater` 在 `QThread::finished` 信号里调用，确保 Worker 在线程结束后才被删除。

---

## 4. QTcpServer / QTcpSocket 实战

### 和原生 Socket 的对比

Day 5–7 用的是操作系统的 socket API（bind/listen/accept/recv/send），Qt 把这些封装成了信号槽驱动的异步模型：

| 原生 Socket | Qt 封装 |
|---|---|
| `bind()` + `listen()` | `QTcpServer::listen()` |
| `accept()` 阻塞等连接 | `newConnection` 信号 |
| `recv()` 阻塞等数据 | `readyRead` 信号 |
| `send()` | `QTcpSocket::write()` |
| 手动管理 fd | Qt 对象树自动管理 |

最大的区别是：原生 Socket 需要 epoll 或多线程来实现非阻塞，Qt 的 Socket 天然是异步的，数据到了发 readyRead 信号，新连接来了发 newConnection 信号，不需要任何额外的并发机制。

### 服务端核心逻辑

```cpp
// 启动监听
m_server->listen(QHostAddress::Any, 9999);

// 新连接到来
void NetDemo::onNewConnection() {
    QTcpSocket *client = m_server->nextPendingConnection();
    connect(client, &QTcpSocket::readyRead, this, &NetDemo::onServerDataReady);
    connect(client, &QTcpSocket::disconnected, this, &NetDemo::onClientDisconnected);
}

// 收到数据，用 sender() 获取触发信号的 Socket
void NetDemo::onServerDataReady() {
    QTcpSocket *client = qobject_cast<QTcpSocket*>(sender());
    QByteArray data = client->readAll();
    client->write("Echo: " + data);  // 回显
}
```

`qobject_cast<QTcpSocket*>(sender())` 是关键技巧：多个 Socket 的 readyRead 连接到同一个槽，用 sender() 获取是哪个 Socket 触发的。

### 客户端断开的处理

```cpp
void NetDemo::onClientDisconnected() {
    QTcpSocket *client = qobject_cast<QTcpSocket*>(sender());
    m_clients.removeOne(client);
    client->deleteLater();  // 不要直接 delete，用 deleteLater
}
```

`deleteLater()` 把删除操作投递到事件循环，等当前信号槽调用链结束后再删除。直接 delete 一个正在发信号的对象会崩溃。这和 Day 7 的 `close(fd)` 后必须 `buffers.erase(fd)` 是同一个道理——资源断开后必须清理关联状态。

---

## 5. 踩过的坑

### find_package 没加 Network 模块

CMakeLists.txt 里 `target_link_libraries` 写了 `Qt::Network`，但 `find_package` 里只有 `Core Widgets LinguistTools`，没有 `Network`。编译报错找不到 `QTcpServer` 头文件。

原因：`target_link_libraries` 只是告诉链接器要链哪个库，但 CMake 得先通过 `find_package` 找到这个库在哪。先 find 才能 link，顺序不能反。

```cmake
# ❌ find 里没写 Network，link 了也没用
find_package(Qt6 6.5 REQUIRED COMPONENTS Core Widgets LinguistTools)
target_link_libraries(app PRIVATE Qt::Core Qt::Widgets Qt::Network)

# ✅ find 和 link 都要写
find_package(Qt6 6.5 REQUIRED COMPONENTS Core Widgets Network LinguistTools)
target_link_libraries(app PRIVATE Qt::Core Qt::Widgets Qt::Network)
```

### Worker 设了 parent 导致 moveToThread 失败

`new FileWorker(this)` 传了 parent，moveToThread 时 Qt 会输出警告并拒绝移动。规则：要 moveToThread 的对象不能有 parent，它的生命周期通过 `QThread::finished → deleteLater` 来管理。

### waitForConnected 会阻塞 UI

客户端用 `m_clientSocket->waitForConnected(3000)` 是阻塞调用，3秒内连不上界面会卡住。正式项目应该用信号：

```cpp
connect(m_clientSocket, &QTcpSocket::connected, this, &NetDemo::onConnected);
m_clientSocket->connectToHost("127.0.0.1", 9999);
// 不阻塞，连接成功后 connected 信号触发
```

Demo 里为了简单用了阻塞方式，正式代码一定要改成异步。

---

## Day 5–7 → Day 10 的知识衔接

| 原生实现（Day 5–7） | Qt 封装（Day 10） |
|---|---|
| pthread_create / std::thread | QThread + moveToThread |
| mutex / condition_variable | QMutex / QWaitCondition（或直接用信号槽） |
| epoll_wait 事件循环 | QThread 内置事件循环 |
| socket / bind / listen / accept | QTcpServer::listen + newConnection 信号 |
| recv / send | readyRead 信号 + readAll / write |
| close(fd) + buffers.erase(fd) | deleteLater() + removeOne() |
| 粘包处理（按行分割） | 仍然需要，Qt 没有自动处理粘包 |

---

## 核心总结

moveToThread 是 Qt 多线程的推荐方式，Worker 不能有 parent，四步走：创建→移动→连接→启动。跨线程信号槽的本质是事件队列投递，子线程 emit 的信号在主线程的事件循环里执行槽函数，所以能安全更新 UI。Qt 的 Socket 是异步信号驱动的，不需要 epoll，但粘包问题仍然存在。find_package 和 target_link_libraries 缺一不可，先 find 才能 link。线程销毁必须 quit + wait，deleteLater 保证对象在安全时机删除。

> **记住**：子线程绝对不能直接操作 UI 控件，只能通过信号槽。这不是建议，是硬性规则——违反必崩溃。理解了跨线程信号槽的事件队列投递机制，就理解了为什么信号槽是线程安全的而直接调用不是。

明天（Day 11）开始项目开发阶段，用原生 C++ Socket 补一个底层网络 Demo，然后正式启动跨平台文件传输工具的开发。

---
