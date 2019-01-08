---
title: 读薄Effective Modern C++ (条款3 理解decltype)
date: 2018-10-19 00:33:48
tags:
- [c++]
categories:
- [编程语言]
- [读书笔记]
---

## 条款3 理解decltype (p23 - p29)
decltype返回一个变量名或表达式的类型。和auto推导相比，它大多数时候只是如实的返回变量或表达式的类型，但是也有一些特殊情况。
<!-- more -->
正常情况下，没有任何意外。
```cpp
const int i = 0;            // decltype(i) is const int

bool f(const Widget& w);    // decltype(w) is const Widget&
                            // decltype(f) is bool(const Widget&)
struct Point{
    int x, y;               // decltype(Point::x) is int
};

Widget w;                   // decltype(w) is Widget

if (f(w)) ...               // decltype(f(w)) is bool

template<typename T>        // simplified version of std::vector
class vector {
public:
    ...
    T& operator[](std::size_t index);
    ...
};

vector<int> v;              // decltype(v) is vector<int>
...
if(v[0] == 0)               // decltype(v[0]) is int&f
```

decltype在我们无法显式的写出某个表达式的类型时候特别有用，
例如我们要写一个获取列表容器在某个位置上的元素的泛型函数，我们无法知道这个容器内部的元素是什么类型，c++11中，我们可以借助尾置返回语法写出以下代码，
```cpp
template<typename Container, typename Index>    // final
auto                                            // C++11
access(Container&& c, Index i)                  // version
-> decltype(std::forward<Container>(c)[i]
{
    return std::forward<Container>(c)[i];
}
```
c++14中我们可以写成，
```cpp
template<typename Container, typename Index>
decltype(auto) access(Container&& c, Index i)
{
    return std::forward<Container>(c)[i];
}
```
代表用decltype的规则来进行类型推导。
该语法不止可以用于函数的返回类型推导，还可以用于变量的类型推导。
```cpp
Widget w;
const Widget& cw = w; // cw的类型为const Widget&
auto myWidget1 = cw;  // auto类型推导，myWidget1的类型为Widget
decltype(auto) myWidget2 = cw; // decltype类型推导，myWidget2的类型为const Widget&
```

需要注意的情况是，decltype会将具有类型T的左值表达式的类型推导为T&，
```cpp
decltype(auto) f1()
{
    int x = 0;
    ...
    return x;        // f1返回类型为int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);     // f2返回类型为int&
}
```
f2返回了一个局部变量的引用！这当然是我们不愿意看到的。        

### 总结         
- decltype几乎总是不加修改的返回变量或表达式的类型
- 对于类型为T并且不是单纯变量名的左值表达式，decltype总是返回T&
- C++14支持decltype(auto)语法，使用decltype的规则进行类型推导



