---
title: 读薄Effective Modern C++ (条款23 理解std::move和std::forward)
date: 2019-02-18 23:33:41
tags:
- [C++]
- [Move semantic]
- [Value category]
- [Rvalue]
categories:
- [PL, Tips]
- [Memo]
---

## 读薄Effective Modern C++ (条款23 理解std::move和std::forward) 

移动语义(Move Semantic)，表达的是资源所有权的转移。通过移动语义，使得C++能够通过移动（转移所有权）的方式来构造对象，它的目的是用于避免（某些时候）更为昂贵的拷贝构造。std::move用于帮助实现移动构造。
std::forward则用于实现完美转发(Perfect Forwarding)，使得参数在传递过程中左值或右值的属性(value category)能够得到保持。

实际上，std::move不移动任何东西，std::forward不转发任何东西，在运行期间，它们什么事情都不会做，不会生成一个字节的可执行代码。它们做的只是类型转换，std::move无条件地把参数转换为右值，而std::forward在只在特定条件下才会执行右值转换。
<!-- more -->
一个接近但不完全符合标准的std::move实现如下,
```cpp
template <typename T>
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType = typename remove_reference<T>::type&&;
    return static_cast<ReturnType>(param);
}
```
但是，std::move也不能保证移动一定会发生，比如一个const的右值就不能被移动。另外真正进行移动的是移动构造函数，而不是std::move。如上所示，std::move所做的仅仅是将参数转换为右值。

而std::forward的最常见用法是在一个接受通用引用参数的模板函数中，把参数“转发”给另一个函数，

```cpp
void process(const Widget& lvalArg);    // 处理左值
void process(Widget&& rvalArg);         // 处理右值

template<typename T>
void logAndProcess(T&& param)
{
     auto now = std::chrono::system_clock::now();   // 获取当前时间

     makeLogEntry("Calling 'process'", now);
     process(std::forward<T>(param));
}
```

logAndProcess函数的参数param被传递给process函数，而process函数针对左值参数和右值参数进行了重载。当我们用左值调用logAndProcess的时候，我们自然是希望把左值转发给process，而当我们用右值调用logAndProcess时，我们希望调用的是右值重载的process。

但是param，作为一个右值引用，其本身是个左值。所以每次只有左值重载的process会被调用。为了防止这样的事情，我们需要一项技术，当且仅当初始化param的实参是右值时，才将param转换为右值。这就是std::forward所做的事了，这也是为什么说std::forward是有条件的类型转换。

## 总结
- std::move表现为无条件的右值转换，其自身不会移动任何东西。
- std::forward仅当参数被右值绑定时，才会把参数转换为右值。
- std::move和std::forward在运行时不做任何事情。
