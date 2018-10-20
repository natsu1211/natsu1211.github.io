---
title: 读薄Effective Modern C++ (Item4 知道如何查看类型推导)
date: 2018-10-20 17:46:42
tags: 
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item4 知道如何查看类型推导 (p30 - p35)
通常来说有三种办法，            

### IDE
在IDE里面的代码编辑器里面当你使用光标悬停在实体之上，常常可以显示出程序实体（例如变量，参数，函数等等）的类型。需要IDE的编辑器对代码的文法有分析的能力。       
对于简单的类型例如int，IDE里面的信息是正常的。对于更加复杂的类型的时候，从IDE里面得到的信息并不一定是有帮助性的。        
<!-- more -->

### 编译器诊断
一个有效的让编译器展示类型的办法就是故意制造编译问题。
首先声明一个类模板，但是并不定义这个模板：

template<typename T>                    // 声明TD
class TD;                               // TD == "Type Displayer"
尝试实例化这个模板会导致错误信息，因为没有模板的实现。想看变量被推导的类型，只要尝试去使用这些类型去实例化TD：           
```cpp
TD<decltype(x)> xType;                  // 引起的错误
TD<decltype(y)> yType;                  // 包含了x和y的类型
```
我使用的变量名字的形式variableNameType是因为这样有利于输出的错误信息可以帮助我定位我要寻找的信息。对上面的代码，我的一个编译器输出了诊断信息，其中的一部分如下：
```cpp
error: aggregate 'TD<int> xType' has incomplete type and cannot be defined
error: aggregate 'TD<const int *> yType' has incomplete type and cannot be defined
```
另一个编译器提供相同的信息，但是格式不太一样：
```cpp
error: 'xType' uses undefined class 'TD<int>'
error: 'yType' uses undefined class 'TD<const int *>'
```
排除格式的区别，基本所有的编译器都会在这种代码的技术中输出有用的错误信息。


### 运行时输出
可以使用，
```cpp
std::cout << typeid(x).name() << '\n'; // display types for
std::cout << typeid(y).name() << '\n'; // x and y
```
这是基于对类似于x或者y运算typeid可以得到一个std::type_info对象，std::type_info有一个成员函数，name可以提供一个C风格的字符串，代表了类型的名字。但是遗憾的是输出通常并不是完全可靠。        
Boost.TypeIndex库可以得到准备的结果，我们可以写一个函数f来查看变量或函数的类型  
```cpp
#include <boost/type_index.hpp>
template<typename T>
void f(const T& param)
{
    using std::cout;
    using boost::typeindex::type_id_with_cvr;

    // show T
    cout << "T = "
         << type_id_with_cvr<T>().pretty_name()
         << '\n';

    // show param's type
    cout << "param = "
         << type_id_with_cvr<decltype(param)>().pretty_name()
         << '\n';
    …
}
```
按照以下方法使用，
```cpp
std::vector<Widget> createVec();    // 工厂方法

const auto vw = createVec();        // init vw w/factory return

if (!vw.empty()) {
    f(&vw[0]);                      // 调用f
    …
}
```
在GNU和Clang的编译器下面，Boost.TypeIndex输出准确的结果：
```cpp
T = Widget const*
param = Widget const* const&
```
微软的编译器实际上输出的结果是一样的：
```cpp
T = class Widget const *
param = class Widget const * const &
```

以上三种方法可以灵活使用。

### 总结
- 类型推导的结果常常可以通过IDE的编辑器，编译器错误输出信息和Boost.TypeIndex库的结果中得到
- 一些工具的结果不一定有帮助性也不一定准确，所以对C++标准的类型推导法则加以理解是很有必要的