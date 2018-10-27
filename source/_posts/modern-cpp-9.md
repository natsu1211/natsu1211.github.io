---
title: 读薄Effective Modern C++ (Item9 优先使用别名声明而不是typedef)
date: 2018-10-27 16:15:51
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item9 优先使用别名声明而不是typedef 

C++11中新增了别名声明，对于之前的typedef语句，可以使用别名声明语法写成
```cpp
// FP等价于一个函数指针，这个函数的参数是一个int类型和
// std::string常量类型，没有返回值
typedef void (*FP)(int, const std::string&);      // typedef

// 同上
using FP = void (*)(int, const std::string&);     // 别名声明
```
也许别名声明看起来可读性更好，但是这并不足以成为放弃typedef使用别名声明的决定性理由。别名声明的真正优势并不在这，而在于它能够定义模板别名，
<!-- more -->

typedef需要新定义一个模板类来模拟实现模板别名功能，
```cpp
template<typename T>                            
struct MyAllocList {                            
    typedef std::list<T, MyAlloc<T>> type;
};

MyAllocList<Widget>::type lw;                   // 客户代码 
```
而使用别名声明则很直接，
```cpp
template<typname T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<Widget> lw;                         // 客户代码
```

特别当你在另一个模板类中使用该类型别名的时候，对比typedef，别名声明语法的优势则显得更加明显，
```cpp
	template<typename T>                            // Widget<T> 包含
	class Widget{                                   // 一个 MyAloocList<T>
	private:                                        // 作为一个数据成员
	  typename MyAllocList<T>::type list;
	  ...
    };
```
因为无法知道名称MyAllocList<T>::type究竟是一个类型，还是一个变量，所以需要加上typename来明确表明这是一个类型。而别名声明则没有冗余的typename和::type，
```cpp
template<typname T>                             
using MyAllocList = std::list<T, MyAlloc<T>>;   // 和以前一样

template<typename T>
class Widget {
private:
	MyAllocList<T> list;  // 编辑器知道MyAllocList<T>一定是一个类型
	...                                          
};
```
另外，对于标准库中的type traits，C++11提供的接口都是基于typedef的，而在C++14中都提供了相应更为简洁的别名声明的版本，
```cpp
std::remove_const<T>::type                // C++11: const T -> T
std::remove_const_t<T>                    // 等价的C++14

std::remove_reference<T>::type            // C++11: T&/T&& -> T
std::remove_reference_t<T>                // 等价的C++14

std::add_lvalue_reference<T>::type        // C++11: T -> T&
std::add_lvalue_reference_t<T>            // 等价的C++14
```
它的定义很简单，即使只能用C++11的环境也只需自己简单的定义一下即可，
```cpp
template<class T>
using remove_const_t = typename remove_const<T>::type;

template<class T>
using remove_reference_t = typename remove_reference<T>::type;

template<class T>
using add_lvalue_reference_t = typename add_lvalue_reference<T>::type;
```

## 总结
- typedef不支持模板化，但是别名声明支持
- 模板别名避免了::type后缀，在模板中，typedef还经常需要使用typename前缀
- C++14为C++11中的type traits提供了模板别名