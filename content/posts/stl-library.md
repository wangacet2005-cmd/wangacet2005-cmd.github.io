+++
title = "Day 2：C++ 面试必考——STL 容器底层原理与选型全攻略"
date = 2026-02-27T15:00:00+08:00
draft = false
weight = 2
tags = ["C++", "STL", "容器", "面试突击", "底层原理"]
categories = ["C++底层修炼"]
+++

> **写在前面**：这是我为期 30 天"PC 客户端开发工程师"冲刺计划的第二天。在打通了智能指针与 RAII 之后，今天进入 STL 的核心战场：**容器（Container）**。容器是 C++ 日常开发中使用频率最高的工具，也是面试中"底层原理"考察的重灾区。今天的目标是：不仅会用，更要知道为什么这么用。

STL 容器分为三大类：**序列容器**（`vector`、`list`、`deque`）、**关联容器**（`map`、`set`）和**无序容器**（`unordered_map`、`unordered_set`）。它们的核心差异在于底层数据结构，而底层数据结构直接决定了各种操作的时间复杂度和使用场景。

---

## 1. `vector`：最常用的动态数组

### 底层原理

`vector` 底层是一块**连续的堆内存**，本质是一个会自动扩容的数组。连续内存使它对 CPU 缓存极为友好，随机访问性能达到 O(1)。

### 核心：扩容机制（面试高频！）

这是关于 `vector` 最重要的知识点。当元素数量（`size`）触达当前容量（`capacity`）上限时，`vector` 会触发扩容：

1. 申请约 **2 倍**大小的新内存（不同标准库实现略有差异）
2. 将旧数据**全部拷贝**（或移动）到新内存
3. **释放旧内存**

这意味着扩容是一个 O(n) 操作，且会导致所有已有的迭代器、指针、引用**全部失效**。

```cpp
#include <vector>
#include <iostream>
using namespace std;

int main() {
    vector<int> v = {1, 2, 3};

    // 如果提前知道数量，用 reserve 预分配，避免多次扩容
    v.reserve(100);

    v.push_back(4);         // O(1) 均摊——末尾追加
    v.insert(v.begin(), 0); // O(n) ——头部插入，需要移动所有元素，慢！
    v.erase(v.begin());     // O(n) ——头部删除，同理

    cout << "size: "     << v.size()     << endl; // 当前元素数量
    cout << "capacity: " << v.capacity() << endl; // 当前已分配的容量
}
```

### 适用场景

需要随机访问、末尾频繁增删的场景。**在没有特殊需求时，`vector` 永远是序列容器的首选。**

---

## 2. `list`：双向链表

### 底层原理

`list` 底层是**双向链表**，每个节点分别存储：数据值、前驱指针、后继指针。内存不连续，散落在堆的各处。

### 一个反直觉的结论

| 操作 | `vector` | `list` |
|------|----------|--------|
| 随机访问 `v[i]` | O(1) ✅ | 不支持 ❌ |
| 末尾插入 | O(1) ✅ | O(1) ✅ |
| 头部 / 中间插入 | O(n) | O(1) ✅（已有迭代器时）|
| CPU 缓存友好 | ✅ 极好 | ❌ 极差 |

虽然 `list` 的中间插入是 O(1)，但在现代硬件上，由于指针跳转导致大量 **cache miss**，**`list` 的实际运行速度往往比 `vector` 更慢**。这是一个在面试中非常能展示深度的点。

### 适用场景

频繁在**已知迭代器位置**进行插入删除，且完全不需要随机访问时。

---

## 3. `deque`：双端队列

### 底层原理

`deque`（double-ended queue）底层是**分段连续内存**——由多个固定大小的内存块组成，由一个中央索引数组管理所有块的地址。

这种结构让它同时拥有了：
- 头尾两端 O(1) 的插入删除
- 下标随机访问（略慢于 `vector`，因为需要二级寻址）

```cpp
#include <deque>

deque<int> dq = {2, 3, 4};
dq.push_front(1); // 头部插入 O(1) ✅  —— vector 做不到
dq.push_back(5);  // 尾部插入 O(1) ✅
dq.pop_front();   // 头部删除 O(1) ✅
cout << dq[0];    // 下标访问依然支持
```

> 💡 `std::stack` 和 `std::queue` 的默认底层实现就是 `deque`，它是它们共同的"地基"。

---

## 4. `map` vs `unordered_map`：面试终极必考题 ⭐

这是今天最重要的内容，几乎每次 C++ 面试都会考到。

### 底层结构决定一切

| 特性 | `map` | `unordered_map` |
|------|-------|-----------------|
| 底层结构 | **红黑树** | **哈希表** |
| 查找时间 | O(log n) | O(1) 平均 |
| 最坏查找 | O(log n) | O(n)（哈希冲突严重时）|
| key 是否有序 | **自动有序** ✅ | 无序 |
| 内存占用 | 相对较少 | 相对较多 |

### ⚠️ 面试高频坑点：用 `[]` 访问会自动插入！

使用 `operator[]` 访问一个**不存在的 key** 时，容器会**自动插入一个默认值**。这往往不是我们想要的行为，应该用 `find()` 替代：

```cpp
unordered_map<string, int> scores;
scores["Alice"] = 95;

// ❌ 危险：如果 "Bob" 不存在，会插入 scores["Bob"] = 0，默默修改了容器！
cout << scores["Bob"];

// ✅ 安全：不存在时返回 end()，不修改容器
auto it = scores.find("Bob");
if (it != scores.end()) {
    cout << it->second;
}
```

### 选择原则

- 需要**按 key 有序遍历** → 用 `map`
- 只需要**高性能查找、插入、删除** → 用 `unordered_map`

---

## 5. `set` 和 `unordered_set`

逻辑上和 `map`/`unordered_map` 完全对应，只存 key，没有 value。最常见的两个用途：**去重** 和 **O(1) 或 O(log n) 判断元素是否存在**。

```cpp
#include <set>

set<int> s = {3, 1, 4, 1, 5, 9, 2, 6};
// 自动去重 + 升序排列：{1, 2, 3, 4, 5, 6, 9}

s.insert(7);
s.erase(3);
cout << s.count(4); // 存在返回 1，不存在返回 0
```

---

## 6. 今日实战：手写单词频率统计器

用 `unordered_map` 实现一个统计字符串中单词出现频率，并按频率从高到低排序的程序。

```cpp
#include <iostream>
#include <sstream>
#include <unordered_map>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    string text = "the quick brown fox jumps over the lazy dog the fox";

    unordered_map<string, int> freq;
    string token;

    // istringstream + >> 是 C++ 分割字符串最地道的写法
    istringstream iss(text);
    while (iss >> token) freq[token]++; // 利用 unordered_map 默认初始化为 0 的特性

    // 将 map 内容导入 vector 以便排序
    vector<pair<string, int>> index;
    for (auto& pairs : freq) index.push_back(move(pairs)); // move 避免拷贝

    // 自定义排序：频率降序，频率相同时按字母升序
    sort(index.begin(), index.end(),
         [](const auto& a, const auto& b) {
             if (a.second != b.second) return a.second > b.second;
             return a.first < b.first;
         });

    for (const auto& p : index) {
        cout << p.first << ": " << p.second << endl;
    }
}
```

**输出结果：**
```
the: 3
fox: 2
brown: 1
dog: 1
jumps: 1
...
```

### 这段代码的三个亮点

- `freq[token]++` 一行完成统计，优雅地利用了 `unordered_map` 对缺失 key 自动初始化为 0 的特性。
- `std::move` 将 map 元素转移进 vector，避免了不必要的深拷贝，性能更好。
- lambda 里加入次级排序（字母序）处理了频率相同时的边界情况，体现了严谨性。

---

## 💡 核心总结：容器选型速查

STL 容器的本质是**用数据结构换时间复杂度**。选错容器，代码不会报错，但性能可能差了一个数量级。

| 需求 | 推荐容器 |
|------|---------|
| 通用序列、随机访问 | `vector` |
| 两端频繁插入删除 | `deque` |
| 频繁中间插入删除（已有迭代器）| `list` |
| 有序键值存储 | `map` |
| 高性能键值查找 | `unordered_map` |
| 有序去重 | `set` |
| 快速判断元素是否存在 | `unordered_set` |

---

## 写在最后

面试官问"为什么用 `unordered_map` 而不是 `map`"，背后考察的正是这套底层逻辑。会用容器是入门，能说清楚底层原理才是加分项。

> **记住**：`vector` 是默认选项，只有当它的弱点（头部插入慢、不支持快速查找）成为真正瓶颈时，才考虑换其他容器。不要过早优化，但要知道优化的方向在哪里。
