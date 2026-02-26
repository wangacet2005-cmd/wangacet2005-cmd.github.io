+++
title = "Day 1：C++ 面试必杀——彻底吃透智能指针与手写 shared_ptr"
date = 2026-02-26T21:00:00+08:00
draft = false
tags = ["C++", "智能指针", "面试突击", "源码剖析"]
categories = ["C++底层修炼"]
+++

> **写在前面**：这是我为期 60 天“PC 客户端开发工程师”冲刺计划的第一天。想要拿下硬核的 C++ 岗位，第一步就是告别 C 语言时代的裸指针（Raw Pointer）。今天彻底打通现代 C++ 的核心护城河：**内存管理与智能指针**。

在现代 C++（C++11 及以后）中，手动 `new` 和 `delete` 已经被视为一种“坏味道”（Bad Smell）。核心思想是 **RAII（Resource Acquisition Is Initialization，资源获取即初始化）**：将内存的生命周期绑定到栈对象的生命周期上，利用作用域结束时自动调用析构函数的特性来释放内存。

标准库 `<memory>` 提供了三大核心智能指针，它们各自解决不同的场景痛点。

---
## 1. 独占所有权的霸道总裁：`std::unique_ptr`

`unique_ptr` 核心语义是**“独占”**。同一个时刻，只能有一个 `unique_ptr` 指向该片内存。

*   **特点**：它**禁用了拷贝构造和赋值运算符**。你不能把它 copy 给别人。
*   **转移控制权**：如果非要移交内存管理权，必须使用 `std::move()` 进行所有权的转移（移动语义）。
*   **性能**：几乎等同于裸指针，属于**零开销抽象**，日常开发中只要不共享资源，**首选 `unique_ptr`**。

```cpp
std::unique_ptr<int> p1 = std::make_unique<int>(100);
// std::unique_ptr<int> p2 = p1; // ❌ 编译报错！禁止拷贝！
std::unique_ptr<int> p3 = std::move(p1); // ✅ 成功！p1 变为空，p3 接管内存
---

### 2. 共享所有权的交际花：'std::shared_ptr'

在客户端开发中（比如多个 UI 组件同时引用同一份数据模型），经常需要共享内存。这时就要用到 shared_ptr。
    **核心机制**：引用计数（Reference Counting）。内部维护着一个“控制块（Control Block）”。
    当一个 shared_ptr 被拷贝时，引用计数 +1。
    当一个 shared_ptr 离开作用域被销毁时，引用计数 -1。
    当引用计数归 0 时，自动 delete 释放内存。
⚠️ 面试高频坑点：shared_ptr 是线程安全的吗？
    答： 它的引用计数增减是线程安全的（底层使用了原子操作 std::atomic），但修改它所指向的对象本身并不是线程安全的。如果多个线程同时读写同一个对象，仍需加锁。

---
## 3. 破局者：'std::weak_ptr' 与循环引用

shared_ptr 看起来很完美，但它有一个致命弱点：循环引用（Circular Reference）。
    假设有互相指着的 A 和 B 两个节点：
    A 内部有一个 shared_ptr 指向 B。
    B 内部也有一个 shared_ptr 指向 A。
    导致 A 释放的前提是 B 释放，B 释放的前提是 A 释放……最终双方的引用计数永远降不到 0，内存泄漏由此产生。
破局方案：std::weak_ptr。
    weak_ptr 是一种不控制对象生命周期的智能指针。
    它指向一个由 shared_ptr 管理的对象，但不会增加引用计数。
    在使用前，需要调用 lock() 方法将其提升为 shared_ptr，顺便检查对象是否已经被释放。

---
## ⚔️ 硬核实战：手撕 MySharedPtr（面试必考）

客户端/底层开发面试中，经常要求白板手写一个简化版的 shared_ptr，考察对拷贝构造、赋值运算符重载以及堆内存管理的理解。
以下是我今天手写并测试通过的核心源码：

< > C++
`#include "iostream"

template<typename T>
class demo_shared_ptr {
private:

    T *ptr;          //指向实际数据的指针
    int *ref_count;  //用来计数的指针

    void release() {
        if(ptr) {
            (*ref_count) --;
            if(*ref_count == 0) {
                std::cout << "ref_count :" << *ref_count << std::endl;
                delete ptr;
                delete ref_count;
                ptr = nullptr;
                ref_count = nullptr;
                std::cout << "leave memory!" << std::endl;
            }
        }
    }

public:

    demo_shared_ptr(T *p) : ptr(p) {
        if(p) ref_count = new int (1);
        else ref_count = new int (0);
    }

    demo_shared_ptr(const demo_shared_ptr& other) {
        ptr = other.ptr;
        ref_count = other.ref_count;
        if (ptr) (*ref_count) ++;
    }

    demo_shared_ptr operator=(const demo_shared_ptr& other) {
        if(this == &other) return *this;
        release();
        ptr = other.ptr;
        ref_count = other.ref_count;
        if(ptr) (*ref_count) ++;
        return *this;
    }

    ~demo_shared_ptr() {
        release();
    }

    T& operator*() const { return *ptr; }

    T& operator->() const { return ptr; }

    int user_count() const {
        return ptr ? *ref_count : 0;
    }

};

int main() {

    demo_shared_ptr<int> p1(new int(100));
    std::cout << "p1 count :" << p1.user_count() << std::endl;

    {
        demo_shared_ptr<int> p2 = p1;
        std::cout << "p1 count :" << p1.user_count() << std::endl;
    }

    std::cout << "p1 count after p2 :" << p1.user_count() << std::endl;

    return 0;

}`

💡 核心总结
手写 shared_ptr 最容易踩坑的地方在赋值运算符重载（operator=）。必须严格遵循三步走：
    1.判断是否是自我赋值（如果是，直接 return）。
    2.释放旧资源（自己原来指向的内存，计数得减一）。
    3.指向新资源（接管别人，计数加一）。
