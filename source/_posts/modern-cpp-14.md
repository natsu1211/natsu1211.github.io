---
title: 读薄Effective Modern C++ (条款14 使用noexcept修饰不抛出异常的函数)
date: 2018-12-01 18:49:51
tags:
- [C++]
- [noexcept]
categories:
- [PL, Tips]
- [Memo]
---

## 条款14 使用noexcept修饰不抛出异常的函数
C++98引入了异常规范，用于显式表明该函数可能抛出的异常类型，语法如下
```cpp
double func (char param) throw (int, char, exception);
```
表明该函数可能会抛出int，char或exception类型的异常，如果抛出了其他类型的，该异常无法被catch，而会导致程序异常终止。      
异常规范是如此严格，并且由于难以知道该函数中调用的函数可能会抛出什么类型的异常，导致异常规范难以使用，在C++11中已经被废弃。     
<!-- more -->
C++11认为对函数真正有用的信息是该函数究竟会不会抛出异常，因而引入了noexcept关键字,用于标识函数不会抛出异常。而C++98中需要使用`throw()`,
```cpp
int f(int x) throw(); // no exceptions from f: C++98 style 
int f(int x) noexcept; // no exceptions from f: C++11 style
```
相比于C++98的`throw()`，`noexcept`主要有两个优势，      

1. 能够获得最大限度的编译器优化。标记为noexcept的函数，优化器可能无须保持调用栈,并且也无须保证object按照构造时相反的顺序被析构。而throw()则没有这些被优化的机会。
2. 标准库的一些函数，针对noexcept函数进行了优化，能够获得更好的性能。比如std::vector::push_back函数，为了保证异常安全性，只有在确保元素的move操作符不会抛出异常，亦即被标记为noexcept的时候，才会使用move操作代替copy操作。      

然而需要注意的是noexcept是函数签名的一部分。也就是说，一旦决定使用noexcept标记函数并公开，你将很难再改变它，因为从函数签名中删除noexcept的改动将是破坏性的。    
另外需要注意的是，大多数函数是异常中立(exception-neutral)的，很多函数本身并不抛出异常，但是它内部调用的函数却会抛出异常，这类函数并不适合被标记为noexcept。
最后，因为历史遗留原因，编译器并不会对在noexcept函数中调用非noexcept函数进行报错和警告。比如std名称空间中的C标准库函数就没有标记为noexcept，即使它们的实现并不会抛出异常。

## 总结      
- noexcept是函数接口的一部分，这意味着使用者依赖于它。
- 比起非noexcept函数，noexcept函数更有可能被优化。
- noexcept对move操作，swap函数，析构函数非常重要。
- 大多数函数是异常中立的。

