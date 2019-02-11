---
title: 读薄Effective Modern C++ (条款22 当使用Pimpl Idiom时，在实现文件中定义特殊成员函数)
date: 2019-02-11 18:29:38
tags:
- [C++]
- [Smart pointer]
categories:
- [PL, Tips]
- [Memo]
---

## 条款22 当使用Pimpl Idiom时，在实现文件中定义特殊成员函数
如果你曾经尝试过缩短编译时间，你应该熟悉Pimpl(Pointer to implementation) 惯用法。这项技术通过把类中的成员变量替换成指向一个实现类（或结构体）的指针，成员变量被放进单独的实现类中，然后通过该指针间接获取原来的成员变量。
例如，原本的Widget是这样的：

```cpp
class Widget {      // 在头文件“widget.h”中
public:
    Widget();
    ...
private:
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;     // Gadget是某个用户定义的类型
};
```

因为Widget的成员变量有std::string，std::vector和Gadget，因此widget.h中需要包含std::string,std::vector以及的头文件。     
使用Pimpl惯用法可以将Widget改写成，

```cpp
class Widget {      // 依然在头文件“widget.h”中
public:
    Widget();
    ~Widget();
    ...
private:
    struct Impl;    // 声明实现类
    Impl *pImpl;    // 声明指针指向实现类
};
```

<!-- more -->
因为Widget不再需要知道std::string，std::vector和Gadget的定义，所以Widget的用户不再需要include那些头文件了。那样加快了编译速度，也意味着当这些头文件内容改变时，Widget的用户不会受到影响。     
一个被声明，却没定义的类型称为不完整类型（incomplete type）。Widget::Impl就是这样的类型，不完整类型能做的事情很少，不过可以声明一个指针指向它们，Pimpl Idiom就是利用了这个特性。

```cpp
#include "widget.h"      // 在实现文件“widget.cpp”
#include "gadget.h"
#include <string>
#include <vector>
struct Widget::Impl {
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};
Widget::Widget() : pImpl(new Impl) {}
Widget::~Widget() { delete pImpl; }
```
经过前面几个条款的说明，你会发现使用std::unique_ptr来代替原生指针是个更好的选择。
头文件如下，

```cpp
class Widget {       // 在“widget.h”
public:
    Widget();
    ...
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;   // 用智能指针代替原生指针
};
```
实现文件如下，

```cpp
#include "widget.h"        // 在“widget.cpp”
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {       // 如前
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()                           // 见条款21
: pImpl(std::make_unique<Impl>())          // 借助std::make_unique创建
{}                                         // std::unique_ptr
```

该代码能够编译，但是当用户使用Widget类时就会出现编译错误，
```cpp
#include "widget.h"
Widget w;      // 报错
```

错误信息通常会提示对不完整类型Impl使用了delete，你也许会感到疑惑，毕竟我们根本没有对Impl进行过delete操作。原因在于w被销毁时（例如，离开作用域）所生成的代码，在那个时刻，它的析构函数被调用，而在我们的实现文件中，我们没有声明析构函数。根据编译器生成特殊成员函数的规则（条款17），编译器会为我们隐式生成一个析构函数。在那个析构函数中，Widget成员变量pImpl的析构函数将被调用。pImpl是个std::unique_ptr<Widget::Impl>对象，即一个使用默认删除器的std::unique_ptr，而std::unique_ptr的默认删除器是对原生指针使用delete。但默认删除器通常先会使用static_assert来确保原生指针不会指向不完整类型。因为编译器生成的Widget的析构函数，和其他所有的特殊成员函数一样，都是隐式内联的，在内联发生的时间点（也就是在头文件中），Impl的定义还不可见，也就是说还是个不完整类型，因此static_assert会失败产生错误信息。     
解决这个问题的办法也很简单，就是自己为Widget声明析构函数，并在Impl的定义之后定义该析构函数。    
头文件如下，

```cpp
class Widget {            // 如前，在"widget.h"
public:
    Widget();
    ~Widget();            // 只是声明
    ...
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};
```

定义文件如下，

```cpp
#include "widget.h"              // 如前， 在"widget.cpp"
#include "gadget.h"
#include <string>
#include <vector>

struct Widget::Impl {            // 如前， 定义Widget::Impl
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()         // 如前
: pImpl(std::make_unique<Impl>())
{}

Widget::~Widget() {}     // 定义析构函数
```

不过如果编译器生成的析构函数是正常工作的，可以使用default关键字，

```cpp
//Widget::~Widget() {}  替换为
Widget::~Widget() = default
```

另外，使用Pimpl Idiom的类天生就是支持移动操作的候选人，因此编译器生成的移动操作符合我们的需要：移动类内部的std::unique_ptr。就像条款17所说，声明了Widget析构函数会阻止编译器生成移动操作，所以如果想要支持移动，我们必须声明这些函数，同时也有必要将移动操作的定义移到实现文件中。

```cpp
class Widget {               // 在“widget.h”
public:
    ...                      // 其他函数，和以前一样
    Widget(const Widget& rhs);                 // 只是声明
    Widget& operator=(const Widget& rhs);      // 只是声明
private:
    struct Impl;        // 如前
    std::unique_ptr<Impl> pImpl;
};
```
```cpp
#include "widget.h"            // 在“widget.cpp”
...                           // 其他头文件和以前一样
struct Widget::Impl { ... };        // 如前

Widget::~Widget() = default;      // 其他函数也和以前一样

Widget::Widget(const Widget& rhs)                    // 拷贝构造
: pImpl(std::make_unique<Impl>(*rhs.pImpl))
{}

Widget& Widget::operator=(const Widget& rhs)        // 拷贝赋值
{
    *pImpl = *rhs.pImpl;
    return *this;
}
```

为了实现Pimpl Idiom，std::unique_ptr毫无疑问是更好的选择，因为Widget和Widget::Impl之间的关系是独占所有权关系。但是知道使用std::shared_ptr时的举动也是很有必要的。
```cpp
class Widget {             // 在“widget.h”
public:
    Widget();
    ...                    // 不用声明析构函数和移动操作
private:
    struct Impl;
    std::shared_ptr<Impl> pImpl      // 用的是std::shared_ptr
};
```
std::unique_ptr和std::shared_ptr之间行为的不同源于它们对自定义删除器的支持不同。对于std::unique_ptr，删除器的类型是智能指针类型的一部分，这导致当使用编译器生成的特殊成员函数时，指向的类型必须是完整类型。对于std::shared_ptr，删除器的类型不是智能指针类型的一部分，这在运行时会导致更大的数据结构和更慢的代码，但是当使用编译器生成的特殊成员函数时，指向的类型不需要是完整类型。

## 总结
- Pimpl Idiom通过减少类用户和类实现之间的编译依赖来减少编译时间。
- 对于类型为std::unique_ptr的pImpl指针，在头文件中声明特殊成员函数，但在实现文件中实现它们，即使使用编译器默认实现（default关键字）时也不例外。
- 上一条的建议适用于std::unique_ptr，不适用于std::shared_ptr。