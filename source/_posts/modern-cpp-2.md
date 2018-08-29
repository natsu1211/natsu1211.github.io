---
title: 读薄Effective Modern C++ (Item2)
date: 2018-08-29 01:07:46
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item2 理解auto类型推导（p18-p23）
auto类型推导的规则本质就是模板类型推导，除了一种特殊情况。
auto类型推导和模板类型推导之间存在着直接的对应关系。可以想象编译器为auto语句生成了对应的模板函数，并采用Item1介绍的规则对类型进行推导。

<!-- more -->
具体的对应规则如下，
```cpp
auto x = 27;
const auto cx = x;
const auto& rx = x;
```
对应
```cpp
template<typename T>                // 推导x的类型的
void func_for_x(T param);           // 概念上的模板

func_for_x(27);                     // 概念上的调用：
                                    // param的类型就是x的类型

template<typename T>
void func_for_cx(const T param);    // 推导cx的概念上的模板

func_for_cx(x);                     // 概念调用：param的推导类型就是cx的类型

template<typename T>
void func_for_rx(const T& param);   // 推导rx概念上的模板

func_for_rx(x);                     // 概念调用：param的推导类型就是rx的类型
```
特殊情况：       
使用C++11新加入的统一初始化时，
```cpp
auto x3 = {27};
auto x4{ 27 };
```
相当于声明了一个类型为std::intializer_list<int>的变量，这个变量包含了一个单一的元素27。

对待花括号初始化的行为是auto唯一和模板类型推导不一样的地方。当auto声明变量被使用一对花括号初始化，推导的类型是std::intializer_list的一个实例。但是如果相同的初始化递给相同的模板，类型推导会失败，代码不能编译。
```cpp
auto x = { 11, 23, 9 };             // x的类型是
                                    // std::initializer_list<int>

template<typename T>                // 和x的声明等价的
void f(T param);                    // 模板

f({ 11, 23, 9 });                   // 错误的！没办法推导T的类型
```
但是，如果你明确模板的param的类型是一个不知道T类型的std::initializer_list<T>：
```cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({ 11, 23, 9 });                   // T被推导成int，initList的
                                    // 类型是std::initializer_list<int>
```
所以auto和模板类型推导的本质区别就是auto假设花括号初始化代表的是std::initializer_list，但是模板类型推导却不是。

对于C++11，这就是auto类型推导规则的全部了。但是对于C++14来说，故事还要继续。C++14允许auto表示推导的函数返回值（参看Item3），而且C++14的lambda可能会在参数声明里面使用auto。但是，这里的auto的规则是模板的类型推导，而不是auto的类型推导。所以一个使用auto声明的返回值的函数，返回一个花括号初始化就无法编译。

```cpp
auto createInitList()
{
    return { 1, 2, 3 };             // 编译错误：不能推导出{ 1, 2, 3 }的类型
}
```
在C++14的lambda里面，当auto用在参数类型声明的时候也是如此：

```cpp
std::vector<int> v;
auto resetV =  [&v](const auto& newValue) { v = newValue; }    // C++14
resetV({ 1, 2, 3 });                // 编译错误，不能推导出{ 1, 2, 3 }的类型
```
#### 总结   
- auto类型推导通常和模板类型推导类似，但是auto类型推导假定花括号初始化代表的类型是std::initializer_list，但是模板类型推导却不是这样。     
- auto在函数返回值或者lambda参数里面执行模板的类型推导，而不是通常意义的auto类型推导
