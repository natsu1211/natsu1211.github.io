---
title: 读薄Effective Modern C++（条款24 区分通用引用和右值引用）
date: 2019-02-21 03:23:33
tags:
- [C++]
- [Universal reference]
- [Rvalue reference]
categories:
- [PL, Tips]
- [Memo]
---

## 条款24 区分通用引用和右值引用

T&&并不总是代表T的右值引用(Rvalue reference)，T&&也可能是一个T的通用引用(Universal reference)。    
需要通过T&&所出现的语境来判断。     通用引用在两种语境出现。最常见的就是模板函数参数，
```cpp
template<typename T>
void f(T&& param);        // param是通用引用
```
第二个语境是auto声明:
```cpp
auto&& var2 = var1;       // var2是通用引用
```
<!-- more -->

这两种语境有个共同点，就是出现了类型推导。在模板f中，param的类型需要被推导，而在var2的声明中，var2的类型要被推导。
而下面这些代码不用类型推断，所以它们的形式虽然是T&&，但是却是右值引用，

对于通用引用，类型推断是必需的，并且引用的类型必须是精确的“T&&”, 不能有cv修饰符。

声明为auto&&的变量都是通用引用，因为既发生了类型推断，又有正确的形式（T&&）。它的使用场景通常是用于lambda表达式的参数，
例如，有一个C++14的lambda来记录调用任意函数执行的所需要的时间，
```cpp
auto timeFuncInvocation = 
  [](auto&& func, auto&&... params)          // C++14
  {
    start timer;
    std::forward<decltype(func)>(func)(
      std::forward<decltype(params)>(params)...
      );
    stop timer and record elapsed time;
  };
```
func是个通用引用，可以绑定任何的可执行对象，不论左值还是右值，而params是通用引用参数包，可以绑定任意数目的任意类型对象。因此timeFuncInvocation可以记录几乎所有的函数执行的所需的时间。

## 总结
- 如果一个模板函数的参数类型是T&&而T是要被推断类型，或者一个对象使用auto&&声明，那么这个参数或者对象是个通用引用。
- 如果类型推断的类型不是精确的T&&，或者不发生类型推断，那么T&&将是右值引用。
- 如果用右值初始化通用引用，通用引用相当于右值引用。如果用左值初始化通用引用，通用引用相当于左值引用。