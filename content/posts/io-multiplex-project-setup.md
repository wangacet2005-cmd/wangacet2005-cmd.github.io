+++
title = "Day 11：IO 多路复用面试突击 + 跨平台文件传输项目启动"
date = 2026-03-11T21:00:00+08:00
draft = false
weight = 11
tags = ["C++", "Qt", "epoll", "IOCP", "select", "CMake", "项目实战", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：基础强化阶段结束，从今天开始进入项目实战。上半天补齐 IO 多路复用的面试知识点（select/epoll/IOCP 对比），下半天正式搭建跨平台文件传输工具的项目框架。这个项目会贯穿 Day 11–21，覆盖桌面开发、网络编程、多线程、跨平台——简历上最有力的证明。

---

## 1. IO 多路复用三大模型对比

### select / epoll / IOCP

|  | select | epoll (Linux) | IOCP (Windows) |
|---|---|---|---|
| 模型 | 每次传整个 fd 集合给内核 | 内核维护红黑树，增量注册 | 完成端口，内核完成 IO 后通知 |
| 时间复杂度 | O(n) 遍历所有 fd | O(1) 就绪链表直接返回 | O(1) |
| fd 上限 | 1024（FD_SETSIZE） | 无硬限制 | 无硬限制 |
| 触发方式 | 水平触发 | 支持水平/边缘触发 | 完成通知（Proactor） |
| 拷贝开销 | 每次调用都要把 fd 集合从用户态拷到内核态 | 只在 epoll_ctl 时拷贝一次 | 无需拷贝 |
| 跨平台 | 是（但性能差） | 仅 Linux | 仅 Windows |

### 面试高频追问

**水平触发 vs 边缘触发**：水平触发（LT）只要 fd 有数据可读就一直通知；边缘触发（ET）只在状态变化时通知一次。ET 性能更好但必须一次读完，否则数据残留不会再通知。

**Reactor vs Proactor**：Reactor（epoll）是"内核告诉你数据就绪了，你自己去读"；Proactor（IOCP）是"你告诉内核要读什么，内核读完了通知你"。epoll 是同步非阻塞，IOCP 是真正的异步。

**Qt 底层用的什么**：Linux 下用 epoll，Windows 下用 select（不是 IOCP），macOS 下用 kqueue。Qt 选择了封装层面的统一而不是性能最优。

---

## 2. 项目架构设计

### 功能规划

核心功能五个：主界面（文件列表+进度条+状态显示）、TCP 文件传输、多线程传输不阻塞 UI、断点续传、多文件队列传输。加分项三个：拖拽文件、系统托盘、跨平台编译。

### 目录结构

```
FileTransfer/
├── CMakeLists.txt
├── README.md
├── src/
│   ├── main.cpp
│   ├── ui/           # 界面层
│   │   ├── mainwindow.h / .cpp
│   ├── network/      # 网络层
│   │   ├── transferserver.h / .cpp
│   │   └── transferclient.h / .cpp
│   ├── core/         # 核心逻辑
│   │   ├── filetransferworker.h / .cpp
│   │   └── transferprotocol.h / .cpp
│   └── utils/        # 工具类
│       ├── logger.h / .cpp
│       └── config.h / .cpp
└── docs/
    └── architecture.md
```

分层的好处是职责清晰：ui 层只管界面交互，network 层只管 Socket 收发，core 层处理传输逻辑，utils 层放通用工具。面试问项目架构时，这个分层就是答案。

---

## 3. 传输协议设计

### 固定长度包头 + 变长数据体

Day 7 讲过粘包问题——TCP 是流协议没有消息边界。解决方案是固定 12 字节的 header，里面包含消息类型和 payload 长度，接收端先读 12 字节拿到长度，再精确读取对应字节数的数据。

```cpp
// 12 字节包头：| type(1) | reserved(3) | payloadSize(8) |
struct PacketHeader {
    MessageType type;
    quint8 reserved[3] = {0};
    quint64 payloadSize = 0;
    static constexpr int SIZE = 12;
};
```

消息类型覆盖完整的传输流程：FileRequest（请求发送）、FileAccept（同意接收）、FileReject（拒绝）、FileData（数据块）、FileComplete（完成）、FileCancel（取消）、ResumeRequest（断点续传）。

### 为什么用 QDataStream 序列化

手动拼字节容易出错，QDataStream 自动处理字节序（统一用 BigEndian）、自动序列化 QString 等 Qt 类型。跨平台传输时字节序一致性是必须的——x86 是小端，网络协议约定大端，QDataStream 帮你屏蔽这个差异。

---

## 4. 主窗口骨架

### QMainWindow 三件套

QMainWindow 自带三个区域：工具栏（QToolBar）、中心控件（setCentralWidget）、状态栏（QStatusBar）。不需要自己写布局来安排这三个区域的位置。

```cpp
// 工具栏
auto *toolbar = addToolBar("主工具栏");
toolbar->addWidget(m_addFilesBtn);
toolbar->addSeparator();
toolbar->addWidget(m_serverBtn);
toolbar->addWidget(m_connectBtn);

// 中心：文件列表
setCentralWidget(central);

// 状态栏
statusBar()->addWidget(m_statusLabel, 1);
statusBar()->addPermanentWidget(m_totalProgress);
```

### UserRole 存完整路径

文件列表里显示的是文件名，但后面传输时需要完整路径。用 `setData(path, Qt::UserRole)` 把路径藏在 Model 的 UserRole 里，不影响显示但随时能取出来：

```cpp
m_fileModel->item(row, 0)->setData(path, Qt::UserRole);

// 取的时候
QString path = m_fileModel->item(row, 0)->data(Qt::UserRole).toString();
```

这是 Qt Model/View 的标准用法，面试问到 Model 自定义数据存储时可以拿这个举例。

---

## 5. 踩过的坑

### MinGW 编译器链接 MSVC 编译的 Qt 库

CLion 默认用自带的 MinGW 编译器，但 Qt 库是 MSVC 编译的（`msvc2022_64`）。MinGW 和 MSVC 的 ABI 完全不兼容，编译能过但链接时报一堆 `undefined reference`，所有 Qt 符号都找不到。

错误特征：每个 Qt 类的方法都报 `undefined reference to '__imp_ZN...'`，数量巨大。看到 `__imp_` 开头的未定义符号 + MinGW 的 `ld.exe` 报错，就是编译器和库不匹配。

解决方案：把 CLion 的工具链切换为 Visual Studio。**Settings → Toolchains → 添加 Visual Studio**，指定 VS 的安装路径。核心原则：Qt 库是哪个编译器编译的，项目就必须用同一个编译器。

### MSVC 找不到 C++ 标准库头文件

切换到 MSVC 后，报 `fatal error C1083: 无法打开包括文件: "type_traits"`。这是因为 CLion 没有正确加载 VS 的环境变量，导致 cl.exe 找不到标准库的 include 路径。

最终解决方案是用 Qt Creator 代替 CLion 开发 Qt 项目。Qt Creator 对 MSVC + Qt 的配置是全自动的，打开 CMakeLists.txt 选 Kit 就能编译，不需要手动配置任何环境变量。

### IDE 选择的教训

CLion 是很好的 C++ IDE，但对 Qt 项目的支持需要大量手动配置。Qt Creator 虽然功能不如 CLion 丰富，但和 Qt 的集成是开箱即用的。项目开发阶段时间紧，**用最省事的工具，把时间花在写代码上**。

---

## Day 10 → Day 11 的知识衔接

| Day 10 概念 | Day 11 扩展 |
|---|---|
| QTcpSocket 异步信号驱动 | 设计完整的传输协议（包头+数据体） |
| QThread + moveToThread | 规划 Worker 在子线程执行传输 |
| 跨线程信号槽更新 UI | 主窗口进度条接收子线程进度信号 |
| Qt Network 模块 | find_package + target_link_libraries 缺一不可 |

---

## 核心总结

IO 多路复用面试三板斧：select 是 O(n) 遍历有 fd 上限，epoll 是 O(1) 就绪链表无上限，IOCP 是真正的异步由内核完成 IO。项目架构分四层：ui/network/core/utils，职责清晰。传输协议用固定 12 字节包头解决粘包，QDataStream 统一字节序。编译器必须和 Qt 库匹配，MinGW 不能链接 MSVC 的库。Qt 项目优先用 Qt Creator 开发，省下配环境的时间写代码。

> **记住**：面试问"你项目中遇到的最大挑战"时，编译器和库 ABI 不匹配导致链接失败就是一个真实的好例子。你能讲清楚 MinGW 和 MSVC 的区别、ABI 兼容性问题、以及如何诊断和解决，这比编造一个挑战有说服力得多。

明天（Day 12）开始填充主界面的文件选择功能和 TCP 传输的核心逻辑，从空壳变成真正能传文件的程序。

---
