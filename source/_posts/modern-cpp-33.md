---
title: 读薄Effective Modern C++（条款33 对需要std::forward的auto&&参数使用decltype)
date: 2019-06-15 21:38:39
tags:
- [C++]
- [Lambda]
categories:
- [PL, Tips]
- [Memo]
---

## 条款33 对需要std::forward的auto&&参数使用decltype) 

从C++14开始，lambda可以在参数类型声明处使用auto，表明该lambda是个泛型lambda，参数的类型需要进行推导。这个特性的实现很直截了当：生成的闭包类中的()运算符函数是一个模板。例如，

```cpp
auto f = [](auto x) { return func(normalize(x)); };
```
编译器生成的闭包类的函数调用运算符看起来是这样的,

```cpp
class SomeCompilerGeneratedClassName {
public:
    template<typename T>         
    auto operator()(T x) const          // 关于auto返回类型见条款3
    { return func(normalize(x)); }

    ...              // 闭包类的其他功能
};
```
<!--more-->
在这个例子中，lambda对x做的唯一的一件事就是把它转发给normalized函数。如果normalized区别对待左值和右值，写这个lambda的正确方式是把x完美转发给normalized，所以需要修改成以下这样，

```cpp
auto f = 
    [](auto&& param)
    {
        return 
          func(normalize(std::forward<decltype(param)>(param)));
    };
```

因为当把一个左值传给lambda时，decltype(x)的结果是一个左值引用，而左值引用作为std::forward的类型参数时，表明std::forward返回的是左值（见条款28）。而对于右值，decltype(x)的结果是一个右值引用，而不是一个非引用值，不过它们产生的结果一样。所以无论是左值还是右值，把decltype(x)传递给std::forward都能得到需要的结果。

再加上6个点，就可以接受任意多个参数了，

```cpp
auto f = 
    [](auto&&... params)
    {
        return 
          func(normalize(std::forward<decltype(params)>(params)...));
    };
```

## 总结
- 对需要完美转发的lambda的auto&&参数使用decltype
