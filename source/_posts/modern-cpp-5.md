---
title: 读薄Effective Modern C++ (条款5 优先使用auto而非显式类型声明)
date: 2018-10-20 19:37:28
tags: 
- [C++]
- [Type inference]
categories:
- [PL, Tips]
- [Memo]
---

## 条款5 优先使用auto而非显式类型声明 (p37 - p42)

比起显式声明，auto声明有几点优势。 

1. 能够避免局部变量未初始化的问题
```cpp
int x1;                   // potentially uninitialized
auto x2;                  // error! initializer required
auto x3 = 0;              // fine, x's value is well-defined
```
<!-- more -->
2. 可以表示无法显式写出的类型（只有编译器知道的类型）
```cpp
auto dereUPLess =                           // comparison func.
    [](const std::unique_ptr<Widget>& p1,   // for Widgets
        const std::unique_ptr<Widget>& p2)  // pointed to by
    { return *p1 < *p2};                    // std::unique_ptrs
```
c++14中可以在lambda的参数中使用auto，使得代码更加简洁。
```cpp
  auto derefLess =                            // C++14 comparison
  [](const auto& p1,                          // function for
     const auto& p2)                          // values pointed
  { return *p1 < *p2; };
```
另外，使用auto来保存lambda表达式，比使用std::function更加节省内存，调用效率也更加的高。

3. 可以避免类型截断
```cpp
std::vector<int> v;
...
unsigned sz = v.size();
```
sz的类型实际上不是unsigned，而是std::vector<int>::size_type。这在64位的系统上可能会产生问题，应为unsinged是32位的。

4. 更加易于重构
当变量或返回值使用auto的地方无需任何对应。

auto会使得类型声明不是那么一目了然和，这是很多人犹豫使用auto的原因，但是IDE得辅助能帮我们缓解这些问题。 另外还有一些auto无法做出正确推导的情况。        
auto的使用并不是强制的，如果最终觉得显式声明更加易读可维护，可以继续使用传统的显式声明，但是要记住auto的这些优势来做出选择。        

### 总结
- auto变量一定要被初始化，并且对由于类型不匹配引起的兼容和效率问题有免疫力，可以简单代码重构，一般会比显式的声明类型敲击更少的键盘
- auto变量也受限于条款2和条款6中描述的陷阱

