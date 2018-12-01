---
title: 读薄Effective Modern C++ (Item12 优先使用delete关键字删除函数而不是不实现的私有函数)
date: 2018-11-04 23:17:23
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item12 

类，继承与虚函数是C++实现面向对象的基础。在子类中覆盖掉父类中对应的虚函数，以此来实现多态。override关键字可以用来显式标识虚函数覆盖，表明该函数在子类中有不同于父类的实现。

如果要使用覆盖的函数，必须满足以下条件：
- 父类中的函数被声明为virtual。
- 父类中和子类中的函数名称必须是完全一致的（除了虚析构函数）。
- 父类中和子类中的函数的参数类型必须完全一致。
- 父类中和子类中的函数的常量特性必须完全一致。
- 父类中和子类中的函数的返回值类型和异常声明必须兼容。
C++11中，附加了一个条件，虽然它并不太常用，
- 函数的引用修饰符（reference qualifiers）必须完全一致。

对覆盖函数的这些要求意味着，一个小的错误会产生一个很大不同的结果。在覆盖函数中出现的错误通常并不会引起编译错误，所以不容易察觉。比如以下的例子，
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};
irtual
class Derived: public Base {
    public:
    virtual void mf1(); // 不是const函数
    virtual void mf2(unsigned int x); // 参数类型不是int
    virtual void mf3() &&; // 引用修饰符不是左值引用
    void mf4() const; // 没有声明为virtual
};
```
子类并没有对父类的函数进行覆盖，而是重新定义了新的函数，这显然不是我们期望的。
因为虚函数覆盖是如此容易出错，所以C++11提供了override关键字来显式表示这是一个虚函数覆盖，如果不符合覆盖的条件，则会产生编译错误。
还有一点好处是，override关键字是上下文相关的，只在函数声明的末尾起效。也就是说以下C++98时代的代码不会产生编译错误，
```cpp
class Warning {
public:
...
void override(); // 名为override的函数
};
// 在C++98和C++11中意义相同
```
## 总结
- 使用override来标识函数覆盖。