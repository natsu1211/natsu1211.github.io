---
title: 读薄Effective Modern C++ (Item1 理解模板类型推导)
date: 2018-08-24 01:27:34
tags: 
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item1 理解模板类型推导（p9-p18）
C++98中，类型推导只发生在函数模板中。C++11中类型推导还会发生在大多数auto关键字和decltype表达式出现的地方，C++14还会发生在decltype(auto)。如果对类型推导没有深刻的理解，很难写出高效率的现代C++代码。      

<!-- more -->
对于以下函数模板声明（伪代码），
```cpp
template<typename T>
void f(ParamType param);
```
以及它的调用
```cpp
f(expr)
```
在编译时，编译器使用传递来的expr推导出T和ParamType，通常T和ParamType不是一个类型，因为ParamType通常会带上类型修饰符。例如
```cpp
template<typename T>
void f(const T& param);

int x = 0;
f(x);
// T is int, ParamType is const int&
```
T的类型由expr和ParamType共同来决定，有3种可能，    
1. ParamType是一个指针或者引用，但是不是一个通用引用（Universal Reference）    
2. ParamType是一个通用引用      
3. ParamType不是指针也不是引用     

对于1，如果expr的类型是个引用，忽略引用的部分。然后利用expr的类型和ParamType对比去判断T的类型。    
例：
```cpp
template<typename T>
void f(T& param);           // param是一个引用类型

int x = 27;                 // x是一个int
const int cx = x;           // cx是一个const int
const int& rx = x;          // rx是const int的引用

f(x);                       // T是int，param的类型时int&

f(cx);                      // T是const int，
                            // param的类型是const int&
f(rx);                      // T是const int
                            // param的类型时const int&
```
```cpp
template<typename T>
void f(T* param);           // param是一个指针

int x = 27;                 // 和之前一样
const int *px = &x;         // px是一个指向const int x的指针

f(&x);                      // T是int，param的类型是int*

f(px);                      // T是const int
                            // param的类型时const int*
```

对于2，如果expr是一个左值，T和ParamType都会被推导成左值引用。这是模板类型T被推导成一个引用的唯一情况。如果expr是一个右值，那么就执行“普通”的法则（第一种情况）。Item23和Item24会对这种情况进行详尽的解释。  
例：
```cpp
template<typename T>
void f(T&& param);          // param现在是一个通用的引用

int x = 27;                 // 和之前一样
const int cx = x;           // 和之前一样
const int& rx = x;          // 和之前一样

f(x);                       // x是左值，所以T是int&
                            // param的类型也是int&

f(cx);                      // cx是左值，所以T是const int&
                            // param的类型也是const int&

f(rx);                      // rx是左值，所以T是const int&
                            // param的类型也是const int&

f(27);                      // 27是右值，所以T是int
                            // param的类型是int&&
```

对于3，param是pass-by-value的，这意味着param是的一份全新的拷贝。和之前一样，如果expr的类型是个引用，将会忽略引用的部分。接下来const，volatile修饰符也要忽略掉。这是有道理的，因为传递过来的参数不能修改不代表它的拷贝不能被修改。    
例：
```cpp
int x = 27;                 // 和之前一样
const int cx = x;           // 和之前一样
const int& rx = x;          // 和之前一样

f(x);                       // T和param的类型都是int

f(cx);                      // T和param的类型也都是int

f(rx);                      // T和param的类型还都是int
```

### 注意事项
#### 数组作为参数      
通常情况下数组类型会退化为指针类型，但是在模板函数的参数是数组的引用的时候情况就不同了。
```cpp
template<typename T>
void f(T& param);                   // 引用参数的模板
const char name[] = "J. P. Briggs";
f(name);                            // 传递数组给f
```
T最后推导出来的实际的类型就是数组。类型推导包括了数组的长度，所以在这个例子里面，T被推导成了const char [13]，函数f的参数（数组的引用）被推导成了const char (&)[13]。          

#### 函数作为参数     
函数也会退化为函数指针，
```cpp
void someFunc(int， double);    // someFunc是一个函数
                                // 类型是void(int, double)

template<typename T>
void f1(T param);               // 在f1中 参数直接按值传递

template<typename T>
void f2(T& param);              // 在f2中 参数是按照引用传递

f1(someFunc);                   // param被推导成函数指针
                                // 类型是void(*)(int, double)

f2(someFunc);                   // param被推导成函数引用
                                // 类型是void(&)(int, double)
```


### 总结
- 在模板类型推导的时候，有引用特性的参数的引用特性会被忽略
- 在推导通用引用参数的时候，左值会被特殊处理
- 在推导按值传递的参数时候，const和volatile参数会被视为非const和非volatile
- 在模板类型推导的时候，参数如果是数组或者函数名称，他们会被退化成指针，除非是用在初始化引用类型













