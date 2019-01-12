---
title: 读薄Effective Modern C++ (条款16 使const成员函数成为线程安全函数)
date: 2019-01-11 02:04:44
tags:
- [c++]
categories:
- [编程语言]
- [读书笔记]
---

## 条款16 使const成员函数成为线程安全函数
假如我们在数学领域工作，我们很可能会需要一个表示多项式的类，一个计算多项式的根（零点）的函数是很有必要的，也就是求多项式等于0时各个因子的值。这个函数不会改变多项式，所以我们把它声明为const：
```cpp
class Polynomial 
{
public:
    using RootsType =            // 该数据结构存储
      std::vector<double>;       // 多项式的根
    ...
    RootsType roots() const;
    ...
};
```

计算多项式的根的操作很耗时，我们会想到可以只计算一次，把结果缓存起来以便以后使用：
```cpp
class Polynomial 
{
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        if (!rootsAreValid)  {   // 如果无可用缓存值
           ...                   // 计算根，把结果存在rootVals
           rootsAreVaild = true;
        }
        return rootVals;
    }
private:
    mutable bool rootAreValid{ false };
    mutable RootsType rootVals{};
};
```
从用户角度来看，既然是const函数那么应当是只读的，在多线程环境下不加锁的调用该函数进行读操作是合情合理的。所以我们有必要保证const成员函数是线程安全的。
然而在这个函数中，因为mutable的使用，成员变量rootsAreValid和rootVals都有可能被改变，在多线程环境有data race的风险。

为了保证线程安全，最简单的办法就是使用互斥锁(mutex)，
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;

    RootsType roots() const
    {
        std::lock_guard<std::mutex> g(m);    // 加锁

        if (!roorsAreValid) {
            ...
            rootsAreValid = true;
        }
        return rootVals;
    }                                        // 解锁
private:
    mutable std::mutex m;
    mutable bool rootsAreValid{ false };
    mutable RootsType rootVals{};
};
```
需要注意std::mutex是一个只可移动（move）类型，把m添加到多项式对象会使得该对象不能被拷贝，只能被移动。

在一些情况中，使用互斥锁太过小题大做。例如，如果你需要计数一个成员被调用了多少次，那么用一个std::atomic计数器（原子操作类型，看条款40）将会减少很多开销（实际上是否会减少开销需要看互斥锁的具体实现），

```cpp
class Point {
public:
    ...
    double distanceFromOrigin() const noexcept
    {
        ++callCount;    // 原子递增
        return  std::sqrt((x * x) + (y * y ));
    }
private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
};
```
和std::mutex一样，std::atomic也是只可移动类型，所以Point对象也只可以移动，不能拷贝。
如果只有一个变量或者一个存储单元需求同步，那么使用std::atomic就足够了，但是你有两个或者更多的变量和存储单元需要以一个单元的形式操作，多个原子变量无法保证这点，应该使用互斥锁。

## 总结

- 保证const成员函数的线程安全，除非它们决不会在并发语境中使用。
- 使用std::atomic变量可能比互斥锁提供更好的性能，不过它们只适用于单一变量和单一存储单元。


