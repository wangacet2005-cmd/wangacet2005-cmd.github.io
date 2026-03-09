+++
title = "Day 8：Qt 桌面开发入门——信号槽机制与布局管理"
date = 2026-03-08T21:00:00+08:00
draft = false
weight = 8
tags = ["C++", "Qt", "信号槽", "布局管理", "CMake", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：从今天开始进入 Qt 桌面开发。Day 5–7 的网络编程是底层能力，Qt 是把这些能力包装成用户能用的产品。今天的目标是搭通环境、理解信号槽机制、掌握三种布局管理器。信号槽是 Qt 的灵魂，后面所有的 UI 交互、多线程通信都建立在它之上。

---

## 1. 环境搭建的关键决策

### CMake 而不是 qmake

选 CMake 构建有三个原因：现代 Qt 项目的主流趋势是 CMake，面试官会问构建系统的选型理由，而且 CMake 跨平台能力比 qmake 更强。最小可用的 CMakeLists.txt：

```cmake
cmake_minimum_required(VERSION 3.16)
project(FileTransfer LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)

find_package(Qt6 REQUIRED COMPONENTS Widgets)

add_executable(app main.cpp mywidget.cpp mywidget.h)
target_link_libraries(app PRIVATE Qt6::Widgets)
```

### AUTOMOC 是什么

`CMAKE_AUTOMOC ON` 让 CMake 在编译前自动扫描带 `Q_OBJECT` 宏的头文件，调用 MOC（Meta-Object Compiler）生成信号槽所需的额外 C++ 代码。没有这一步，所有 connect() 调用在运行时都会静默失败。

### MSVC 而不是 MinGW

Windows 下选 MSVC 编译器，原因是工业界 PC 客户端开发几乎都用 MSVC，调试工具链（WinDbg、VS 诊断工具）更成熟，面试时说"我用 MSVC 编译"比"我用 MinGW"更有说服力。

---

## 2. Qt 对象树与内存管理

### 父子关系自动析构

Qt 的内存管理不完全靠智能指针，而是靠对象树。设了 parent 的子对象，parent 析构时会自动 delete 所有子对象。

```cpp
QPushButton *button = new QPushButton("点击我", &window);
// window 销毁时自动 delete button，不需要手动管理
```

这解释了为什么 Qt 代码里到处都是裸 new 却不会泄漏——只要 parent 设对了，对象树会处理一切。但如果你 new 了一个对象没设 parent 也没手动 delete，那就是真泄漏。

### QApplication 和事件循环

`QApplication app(argc, argv)` 创建唯一的应用实例，`app.exec()` 启动事件循环。程序在 exec() 里阻塞，不断从操作系统取事件（鼠标点击、键盘输入、定时器、网络数据），分发给对应的 QObject 处理。这和 Day 7 的 epoll 事件循环是同一个思路——都是"等事件→处理事件→等下一个事件"的无限循环。

---

## 3. 信号槽机制

### 三种连接写法

```cpp
// 写法1：成员函数指针（推荐，编译期类型检查）
connect(button, &QPushButton::clicked, this, &MyWidget::onSendClicked);

// 写法2：Lambda（简单逻辑时方便）
connect(m_clearBtn, &QPushButton::clicked, [this]() {
    m_input->clear();
});

// 写法3：字符串宏（老代码常见，运行时才检查，不推荐）
connect(button, SIGNAL(clicked()), this, SLOT(onButtonClicked()));
```

写法1是首选，类型写错了编译器直接报错。写法3的问题是信号名拼错了编译能过，运行时才在控制台输出警告，很容易漏掉。

### 自定义信号的规则

信号只声明不实现，MOC 自动生成实现代码。用 `emit` 关键字发射信号。一个信号可以连接多个槽（都会被调用），一个槽也可以接收多个信号。

```cpp
signals:
    void messageReady(const QString &msg);  // 只声明

// 在槽函数里发射
emit messageReady(text);
```

### 信号槽的本质

信号槽本质上是观察者模式的实现。connect() 把信号和槽注册到元对象系统的一张表里，emit 触发时遍历这张表依次调用所有连接的槽。如果是同一个线程内，槽函数是直接调用（DirectConnection）；如果跨线程，信号会被包装成事件放进目标线程的事件队列（QueuedConnection），这就和 Day 7 的事件循环衔接上了。

---

## 4. 三种布局管理器

### 核心原则：手写布局，不用 Designer 拖拽

面试时被问"你用 Designer 还是手写"，回答手写。Designer 生成的代码可读性差，出问题很难调试，而且面试官想看的是你理不理解布局逻辑，不是你会不会拖控件。

### QVBoxLayout / QHBoxLayout / QGridLayout

```cpp
// 垂直布局
auto *vbox = new QVBoxLayout;
vbox->addWidget(widget1);
vbox->addWidget(widget2);

// 水平布局
auto *hbox = new QHBoxLayout;
hbox->addWidget(widget1);
hbox->addWidget(widget2);

// 网格布局
auto *grid = new QGridLayout;
grid->addWidget(widget1, 0, 0);        // 第0行第0列
grid->addWidget(widget2, 1, 0, 1, 2);  // 第1行，跨2列
```

### 布局嵌套是常态

实际开发中几乎不会只用一种布局，而是嵌套组合。典型结构：

```cpp
auto *mainLayout = new QVBoxLayout(this);
mainLayout->addLayout(hbox);        // 上面放水平布局（输入框+按钮）
mainLayout->addWidget(table, 1);    // 中间放表格，stretch=1 占满剩余空间
mainLayout->addStretch();           // 弹性空白
mainLayout->addWidget(statusBar);   // 底部状态栏
```

`addStretch()` 插入弹性空白，把后面的控件推到底部。`addWidget(widget, 1)` 的第二个参数是拉伸因子，值越大分到的空间越多。

---

## 5. 踩过的坑

### Q_OBJECT 宏漏了

头文件里忘写 `Q_OBJECT`，编译能过但所有 connect() 静默失败，界面上点按钮没反应。这个 bug 排查起来很痛苦，因为没有任何报错信息。规则：只要类里用了 signals/slots/connect，就必须加 Q_OBJECT。

### 头文件没 include 导致 "undeclared identifier"

`mywidget.cpp` 忘了 `#include "mywidget.h"`，所有 `MyWidget::` 开头的函数都报 "Use of undeclared identifier 'MyWidget'"。报错信息很明确，但新手容易在复制代码时漏掉 include。

### 头文件必须加到 CMakeLists.txt

`add_executable(app main.cpp mywidget.cpp mywidget.h)` 里的 `.h` 文件不能省。AUTOMOC 需要扫描头文件才能找到 Q_OBJECT 宏，如果没加进去，MOC 不会处理这个文件，链接时会报符号找不到。

---

## Day 7 → Day 8 的知识衔接

| Day 7 概念 | Day 8 对应 |
|---|---|
| epoll 事件循环 | QApplication::exec() 事件循环 |
| fd 对应独立缓冲区 | QObject 对象树管理各自的状态 |
| 观察者模式（fd 注册到 epoll） | 信号槽注册到元对象系统 |
| 事件驱动（epoll_wait 等事件） | 事件驱动（事件循环等 QEvent） |

---

## 核心总结

CMake + AUTOMOC 是现代 Qt 项目的标配，Q_OBJECT 宏不能忘且头文件必须加到 CMakeLists.txt。信号槽用成员函数指针写法，编译期检查类型安全。布局管理器手写不拖拽，嵌套组合是常态。Qt 的对象树负责内存管理，设了 parent 就不用手动 delete。信号槽本质是观察者模式，跨线程时自动切换为事件队列投递，和 epoll 事件循环是同一个思路。

> **记住**：Qt 代码里到处裸 new 不是因为不管内存，而是对象树在兜底。但如果 new 了对象既没设 parent 也没手动 delete，那就是真泄漏——后面用 Valgrind 扫的时候能抓出来。

明天（Day 9）进入 Qt 常用控件和事件系统，重点是 Model/View 架构和事件分发流程，这是后面做文件传输项目的直接基础。

---
