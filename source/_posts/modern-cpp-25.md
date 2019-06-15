---
title: 读薄Effective Modern C++（条款25 对右值引用使用std::move，对通用引用使用std::forward)
date: 2019-03-28 00:48:34
tags:
- [C++]
- [Universal reference]
- [Rvalue reference]
categories:
- [PL, Tips]
- [Memo]
---

## 条款25 对右值引用使用std::move，对通用引用使用std::forward
右值引用只能绑定到能够被移动的对象，所以当看到一个右值应用作为函数参数时，你可以也应当对其使用std::move来获得更好的效率。

而条款24告诉我们，通用引用有可能绑定到能够被移动的对象。条款23说明了这是std::forward做的事情，

对右值引用使用std::forward会表现出正确的行为，但是源代码会是冗长易错,并且不符合语言习惯的，因此应该避免对右值引用使用std::forward。真正糟糕的是对通用引用使用std::move，因为它移动的对象有可能是个左值，比如局部变量，这将引起调用者难以察觉的bug。
<!-- more -->
```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)       // newName是个通用引用
    { name = std::move(newName); }  // 可以编译，不过太糟了！太糟了！
    ...

private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName(); // 工厂函数

Widget w;

auto n = getWidgetName(); // n是局部变量

w.setName(n); // 把n移动到w

...             // 到了这里n的值是未知的
```
虽然可以通过左值和右值进行重载来替代通用引用，
```cpp
class Widget {
public:
    void setName(const std::string& newName)
    { name = newName; }

    void setName(std::string&& newName)
    { name = std::move(newName); } 

    ...
};
```

但是比起通用引用作为参数，

1. 它有更多的源代码要写和维护（用了两个函数替代一个模板函数）

2. 它效率更低。因此w里的name成员变量直接由字符串赋值构造，没有产生std::string临时对象。而在重载的setName版本中，会创建一个临时对象（用const char* 来创建std::string），再把这个临时对象移动到w的成员变量中。因此，调用一次setName需要执行一次std::string的构造函数，和一次std::string的移动赋值运算符函数，还有一次std::string的析构函数。

3. 使用两个重载函数的最严重的问题是这种设计的可扩展性差。如果函数有更多的参数，每个参数都可以是左值或右值，那么重载函数的数量会成几何增加：n个参数需要2^n个重载。另外一些模板函数可以接受无限个参数，每个都可以是左值和右值。比如std::make_shared和std::make_unique。

最后，不应当对函数内部被返回的局部变量使用std::move,因为它会阻碍编译器进行RVO优化，且不会带来其他任何好处。

## 总结
- 在最后一次使用右值引用或者通用引用时，对右值引用使用std::move，对通用引用使用std::forward。
- 在一个通过值返回的函数中，如果返回的是右值引用对其使用std::move，如果是通用引用则应该使用std::forward）。
- 如果局部变量有资格进行返回值优化（RVO），不要对它们使用std::move或std::forward。
