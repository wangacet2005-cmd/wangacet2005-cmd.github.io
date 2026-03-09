+++
title = "Day 9：Qt 控件与事件系统——Model/View 架构与事件分发"
date = 2026-03-09T21:00:00+08:00
draft = false
weight = 9
tags = ["C++", "Qt", "Model/View", "事件系统", "QSS", "QTableView", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：Day 8 打通了信号槽和布局，今天往上搭两层：用 Model/View 架构管理数据展示，用事件系统处理键盘鼠标交互。这两个是后面做文件传输项目的直接基础——文件列表用 Model/View，快捷键操作用事件重写，界面风格用 QSS。

---

## 1. Model/View 架构

### 为什么不直接往表格里塞数据

Qt 早期有 QTableWidget 这种"数据和控件绑死"的设计，用起来简单但扩展性差。Model/View 把数据和显示彻底分离：

```
Model（数据层）  ←→  View（显示层）
       ↑
    Delegate（渲染/编辑层，可选）
```

**Model** 负责存数据，暴露 `data()`、`rowCount()`、`columnCount()` 等标准接口。**View** 从 Model 拿数据渲染，自己不存任何业务数据。**同一个 Model 可以接多个 View**，改一次数据所有 View 自动刷新。

### QStandardItemModel + QTableView 实战

```cpp
m_model = new QStandardItemModel(0, 3, this);
m_model->setHorizontalHeaderLabels({"文件名", "大小", "状态"});

m_tableView = new QTableView(this);
m_tableView->setModel(m_model);
m_tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
m_tableView->horizontalHeader()->setStretchLastSection(true);
```

添加数据用 `appendRow`，每个单元格是一个 `QStandardItem`：

```cpp
m_model->appendRow({
    new QStandardItem(name),
    new QStandardItem(QString::number(size) + " KB"),
    new QStandardItem("等待中")
});
```

删除选中行要从后往前删，否则索引错位：

```cpp
auto selected = m_tableView->selectionModel()->selectedRows();
for (int i = selected.size() - 1; i >= 0; i--) {
    m_model->removeRow(selected[i].row());
}
```

### 面试怎么讲 Model/View

三句话说清楚：数据和显示分离，Model 管数据、View 管渲染；改 Model 的数据 View 自动更新，不需要手动刷新界面；同一个 Model 可以同时驱动表格、列表、树形等不同 View。

---

## 2. 事件系统

### Qt 事件分发流程

```
操作系统事件
  → QApplication::notify()    // 入口
    → QObject::event()        // 按事件类型分发
      → keyPressEvent()       // 具体处理函数
      → mousePressEvent()
      → paintEvent()
      → ...
    → 未处理则传给 parent widget
```

关键链路：`notify() → event() → 具体 Handler → 父控件`。在 Handler 里如果不调父类实现（如 `QWidget::keyPressEvent(event)`），事件就不会继续向上传递。

### 重写事件处理函数

在头文件声明 `protected` 的 override 函数：

```cpp
protected:
    void keyPressEvent(QKeyEvent *event) override;
    void mousePressEvent(QMouseEvent *event) override;
```

实现里既做自己的逻辑，又要调父类保证事件链不断：

```cpp
void ControlsDemo::keyPressEvent(QKeyEvent *event) {
    if (event->key() == Qt::Key_Delete) {
        onDeleteRow();
    } else if (event->key() == Qt::Key_Return
               && event->modifiers() == Qt::ControlModifier) {
        onAddRow();
    }
    QWidget::keyPressEvent(event);  // 不调这行，事件链断裂
}
```

### 事件 vs 信号槽

事件和信号槽不是竞争关系，而是不同层级。事件是底层机制，由操作系统产生，经过 Qt 事件循环分发；信号槽是上层机制，由程序员在代码里主动 connect。很多信号的底层就是事件触发的——比如 QPushButton 的 `clicked()` 信号，底层是 `mousePressEvent` + `mouseReleaseEvent` 组合判断出来的。

---

## 3. 搜索过滤的实现

### 实时过滤不需要重建 Model

不需要每次搜索都重新填充 Model，直接用 `setRowHidden` 控制行的可见性：

```cpp
void ControlsDemo::onFilterChanged(const QString &text) {
    for (int i = 0; i < m_model->rowCount(); i++) {
        bool match = m_model->item(i, 0)->text()
                     .contains(text, Qt::CaseInsensitive);
        m_tableView->setRowHidden(i, !match);
    }
}
```

连接到 QLineEdit 的 `textChanged` 信号，用户每输入一个字符就触发一次过滤。数据量大的时候可以换成 QSortFilterProxyModel，它在 Model 和 View 之间加了一层代理，过滤逻辑更干净，但对于当前规模直接隐藏行就够了。

---

## 4. QSS 样式表

### 语法和 CSS 几乎一样

选择器选控件类型，伪状态用 `:hover`、`:pressed`、`:focus`，子控件用 `::chunk`、`::section`：

```css
QPushButton {
    background: #2E86C1;
    color: white;
    padding: 6px 16px;
    border-radius: 3px;
}
QPushButton:hover { background: #2471A3; }
QPushButton:pressed { background: #1A5276; }

QHeaderView::section {
    background: #2C3E50;
    color: white;
    font-weight: bold;
}

QProgressBar::chunk {
    background: #27AE60;
    border-radius: 2px;
}
```

### 加载方式

两种方式：代码内写死（适合 Demo），或者加载外部 .qss 文件（适合正式项目）：

```cpp
// 方式1：代码内
setStyleSheet("QPushButton { background: #2E86C1; }");

// 方式2：外部文件（推荐）
QFile f(":/styles/app.qss");
f.open(QFile::ReadOnly);
qApp->setStyleSheet(f.readAll());
```

正式项目用方式2，设计师改样式不用重新编译。

---

## 5. 踩过的坑

### qrand() 在 Qt 6 中被移除

Qt 5 的 `qrand()` 在 Qt 6 里不存在了，编译报 "undeclared identifier 'qrand'"。替换为：

```cpp
#include <QRandomGenerator>
QRandomGenerator::global()->bounded(1000, 10000)
```

`bounded()` 返回 int，不能直接当 QStandardItem 用，必须包一层：

```cpp
new QStandardItem(QString::number(
    QRandomGenerator::global()->bounded(1000, 10000)) + " KB")
```

### 声明了 override 函数但忘了写实现

头文件里声明了 `keyPressEvent` 和 `mousePressEvent` 的 override，但 .cpp 里没写函数体，链接时报 LNK2001 "无法解析的外部符号"。规则：override 声明了就必须实现，哪怕函数体只有一行调用父类。

### 删除行必须从后往前

选中了第 2、5、8 行要删除，如果从前往后删：删完第 2 行后原来的第 5 行变成了第 4 行，原来的第 8 行变成了第 7 行，后续 removeRow 的索引全错了。从后往前删（8→5→2），前面的行索引不受影响。

---

## Day 8 → Day 9 的知识衔接

| Day 8 概念 | Day 9 扩展 |
|---|---|
| 信号槽连接按钮点击 | 信号连接 textChanged 实现实时过滤 |
| QVBoxLayout 单层布局 | 布局嵌套 + stretch 控制空间分配 |
| 对象树管理内存 | Model 管理数据，View 管理显示，各司其职 |
| QSS 切换主题色 | QSS 完整美化表格、按钮、进度条 |

---

## 核心总结

Model/View 架构是数据和显示分离，改 Model 数据所有 View 自动更新，面试讲清楚这一点就够了。事件分发的链路是 notify → event → 具体 Handler → 父控件，重写事件处理函数必须调父类实现否则事件链断裂。删除多行必须从后往前，否则索引错位。qrand 在 Qt 6 已移除，用 QRandomGenerator 替代。QSS 和 CSS 语法几乎一样，正式项目用外部 .qss 文件加载。

> **记住**：事件是底层（操作系统产生），信号槽是上层（程序员 connect）。QPushButton 的 clicked 信号底层就是鼠标事件处理出来的。理解这个层级关系，面试问"事件和信号槽的区别"就能答得很清楚。

明天（Day 10）进入 Qt 多线程与网络模块，把 Day 5–7 的原生 Socket 和多线程知识用 Qt 的方式重新实现——QThread、QTcpSocket、跨线程信号槽。

---
