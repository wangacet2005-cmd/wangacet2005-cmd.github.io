+++
title = "Day 3：C++ 面试必考——STL 算法与迭代器全面解析"
date = 2026-02-28T15:00:00+08:00
draft = false
weight = 3
tags = ["C++", "STL", "算法", "迭代器", "面试突击"]
categories = ["C++底层修炼"]
+++

> **写在前面**：这是我为期 30 天"PC 客户端开发工程师"冲刺计划的第三天。掌握了 STL 容器之后，今天进入与容器配套的另一半：**算法与迭代器**。STL 算法的设计哲学是"与容器解耦"——算法不关心数据装在什么容器里，只通过迭代器操作数据。理解这一点，才能真正用好 STL。

---

## 1. 迭代器：算法与容器之间的桥梁

迭代器是一个**行为像指针的对象**。它屏蔽了容器的内部实现细节，让算法可以统一地操作任意容器中的数据。

### 五种迭代器类型

| 类型 | 支持操作 | 代表容器 |
|------|---------|---------|
| 输入迭代器 | 只读、单向 | `istream` |
| 输出迭代器 | 只写、单向 | `ostream` |
| 前向迭代器 | 读写、单向 | `unordered_map` |
| 双向迭代器 | 读写、双向 | `list`、`map` |
| 随机访问迭代器 | 读写、随机跳跃 | `vector`、`deque` |

能力越强的迭代器，兼容能力越强——需要双向迭代器的地方，传入随机访问迭代器完全没问题，反之则会编译报错。

```cpp
vector<int> v = {1, 2, 3, 4, 5};

auto it  = v.begin(); // 指向第一个元素
auto end = v.end();   // 指向最后一个元素的下一位（哨兵）

cout << *it;  // 解引用：输出 1
++it;         // 移动到下一个
it += 2;      // 随机访问迭代器专属：跳跃式移动
```

### ⚠️ 面试高频坑点：`end()` 不能解引用

`end()` 指向的是**最后一个元素的下一位**，是一个"哨兵"位置，并不指向任何有效元素。对它解引用是未定义行为，轻则输出乱码，重则程序崩溃。

```cpp
// ❌ 绝对不能这样做
cout << *v.end();

// ✅ 正确：先判断，再解引用
auto it = find(v.begin(), v.end(), 99);
if (it != v.end()) cout << *it;
```

---

## 2. 常用 STL 算法详解

所有算法都在 `<algorithm>` 头文件中，`accumulate` 在 `<numeric>` 中。

### `sort`：排序

```cpp
vector<int> v = {5, 3, 1, 4, 2};

sort(v.begin(), v.end());                      // 升序
sort(v.begin(), v.end(), greater<int>());      // 降序
sort(v.begin(), v.end(), [](int a, int b) {    // 自定义规则
    return a > b;
});
```

### `find` / `find_if`：查找

```cpp
// 查找特定值
auto it = find(v.begin(), v.end(), 3);

// 按条件查找（更常用）
auto it2 = find_if(v.begin(), v.end(), [](int x) {
    return x > 3; // 找第一个大于 3 的元素
});

if (it2 != v.end()) cout << *it2;
```

### `transform`：变换每个元素 ⭐

`transform` 是今天最重要的算法，也是最容易写错的。它有两种形式：

```cpp
vector<int> v = {1, 2, 3, 4, 5};

// 一元操作：对每个元素做变换，结果写回原容器（原地变换）
transform(v.begin(), v.end(), v.begin(), [](int x) {
    return x * 2;
});
// v = {2, 4, 6, 8, 10}

// 二元操作：两个容器对应元素合并
vector<int> a = {1, 2, 3};
vector<int> b = {10, 20, 30};
vector<int> result(3);
transform(a.begin(), a.end(), b.begin(), result.begin(), [](int x, int y) {
    return x + y;
});
// result = {11, 22, 33}
```

> 💡 `transform` 的目标容器必须提前分配好空间，否则会写入非法内存导致崩溃。

### `count_if`：按条件计数

```cpp
int fail = count_if(v.begin(), v.end(), [](int x) {
    return x < 60;
});
```

### erase-remove 惯用法：删除满足条件的元素 ⭐

这是 STL 里最经典的惯用法，面试中几乎必考。`remove_if` 并不真正删除元素，只是把"不需要删除"的元素移到容器前面，返回新的逻辑末尾。真正的删除由后面的 `erase` 完成。

```cpp
// ❌ 错误：remove_if 不能单独使用，元素数量不变，只是被移走了
remove_if(v.begin(), v.end(), [](int x) { return x < 60; });

// ✅ 正确：erase-remove 组合拳
v.erase(
    remove_if(v.begin(), v.end(), [](int x) { return x < 60; }),
    v.end()
);
```

---

## 3. 今日实战：学生成绩处理器

不使用任何显式 `for`/`while` 循环，只用 STL 算法完成对一组学生成绩的全套处理。

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {
    vector<int> scores = {85, 92, 60, 78, 95, 55, 88, 70, 40, 99};

    // 1. 计算平均分
    double avg = (double)accumulate(scores.begin(), scores.end(), 0) / scores.size();
    cout << "平均分：" << avg << endl;

    // 2. 最高分 / 最低分（max_element 返回迭代器，需解引用）
    cout << "最高分：" << *max_element(scores.begin(), scores.end()) << endl;
    cout << "最低分：" << *min_element(scores.begin(), scores.end()) << endl;

    // 3. 统计不及格人数
    int fail = count_if(scores.begin(), scores.end(), [](int x) { return x < 60; });
    cout << "不及格人数：" << fail << endl;

    // 4. 所有成绩 +5，但不超过 100（原地变换）
    transform(scores.begin(), scores.end(), scores.begin(), [](int x) {
        return min(x + 5, 100);
    });

    // 5. 删除仍不及格的成绩（erase-remove 惯用法）
    scores.erase(
        remove_if(scores.begin(), scores.end(), [](int x) { return x < 60; }),
        scores.end()
    );

    // 6. 降序输出
    sort(scores.begin(), scores.end(), [](int a, int b) { return a > b; });
    for_each(scores.begin(), scores.end(), [](int x) { cout << x << " "; });
}
```

**输出结果：**
```
平均分：76.2
最高分：99
最低分：40
不及格人数：2
100 100 97 93 83 75 65 65
```

---

## 4. 今日踩坑复盘

**坑一：`accumulate` 只是求和，平均分要自己除**

`accumulate` 返回的是总和，要得到平均分还需要除以 `scores.size()`，并注意将结果转为 `double` 避免整数截断。

**坑二：`max_element` 返回的是迭代器，不是值**

和 `find` 一样，`max_element` / `min_element` 返回迭代器，必须用 `*` 解引用才能拿到实际值。

**坑三：`transform` 的目标容器必须预先分配空间**

```cpp
// ❌ result 是空的，写入会崩溃
vector<int> result;
transform(v.begin(), v.end(), result.begin(), ...);

// ✅ 提前分配空间
vector<int> result(v.size());
transform(v.begin(), v.end(), result.begin(), ...);

// ✅ 或者直接原地变换
transform(v.begin(), v.end(), v.begin(), ...);
```

**坑四：`sort` 的 lambda 参数不需要引用**

```cpp
// ❌ 多此一举
sort(v.begin(), v.end(), [](int& a, int& b) { return a > b; });

// ✅ 值传递即可
sort(v.begin(), v.end(), [](int a, int b) { return a > b; });
```

---

## 💡 核心总结

STL 算法的本质是**把"做什么"和"对谁做"分离**。迭代器负责指定范围，lambda 负责定义规则，算法负责执行逻辑。三者组合在一起，可以写出极为简洁、表达力强的代码。

> **记住**：`remove_if` 不删除元素，它只是"移走"；真正的删除必须配合 `erase`。这个 erase-remove 组合拳是 STL 使用中最经典的惯用法，务必熟记。

---
