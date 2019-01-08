---
title: 读薄Effective Modern C++ (条款6 当auto推导出非预期类型时应当使用显式的类型初始化)
date: 2018-10-21 17:59:29
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---

## 条款6 当auto推导出非预期类型时应当使用显式的类型初始化
使用auto声明来接受一个返回代理类型的表达式，往往会偏离我们的意图。      
<!-- more -->
例如假设有一个函数接受一个Widget返回一个std::vector<bool>，其中每个bool表征Widget是否接受一个特定的特性：
```cpp
std::vector<bool> features(const Widget& w);
```
进一步的，假设第五个bit表示Widget是否有高优先级。我们可以这样写代码：
```cpp
Widget w;
…
bool highPriority = features(w)[5];         // w是不是高优先级的？
…
processWidget(w, highPriority);             // 配合优先级处理w
```

但是，如果写成
```cpp
auto highPriority = features(w)[5];         // w是不是高优先级的？
```
就会出现问题，此时highPriority的类型并不是`bool`，而是`std::vector<bool>::reference`。之前的代码能够工作是由于存在`std::vector<bool>::reference`到`bool`的隐式转换。

也就是说，我们要避免使用下面的代码的形式：
```
auto someVar = expression of "invisible" proxy class type;
```
这种时候应该做一个显式的类型转换。
```cpp
auto highPriority = static_cast<bool>(features(w)[5]);
```

### 总结
- 代理类型会导致auto推导出错误的类型      
- 显式的强制类型转换能够使auto推导出期望的类型