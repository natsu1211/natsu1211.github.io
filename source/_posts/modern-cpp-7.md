---
title: 读薄Effective Modern C++ (Item7 区分()与{}初始化)
date: 2018-10-21 19:57:37
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item7 区分()与{}初始化
c++11新增了使用{}进行初始化的统一初始化语法， 与c++98的()并存，应该使用哪种初始化常常让初学者困惑，因此有必要弄清楚它们之间的区别。
<!-- more -->
首先，初始化一个变量一般有4种方法，
```cpp
int x(0);     // ()初始化
int y = 0;    // =初始化
int z{0};     // {}初始化
int z = {0};  // ={}初始化，可以认为和{}没有区别
```
统一初始化可以用来直接初始化std容器，原子变量，非静态类成员变量，这些都是()初始化做不到的。
```cpp
std::vector<int> v{ 1, 3, 5 }; 
```
```cpp
std::atomic<int> ai1{ 0 }; //没问题
std::atomic<int> ai2(0);   //没问题
std::atomic<int> ai3 = 0;  //错误！
```
```cpp
class Widget {
    ...
private:
    int x{ 0 }; //没问题
    int y = 0;  //没问题
    int z(0);   //错误！
};
```


另外，统一初始化可以避免()初始化的一些问题。

能够避免类型截断，
```cpp
double x, y, z;
int sum1{ x + y + z }; // error!
int sum2(x + y + z); // okay (截断为int)
int sum3 = x + y + z; // okay
```
还能够避免most vexing parse（能够解释为函数声明的表达式就认为它是函数声明），
```cpp
Widget w1(10); //调用Widget的构造函数，参数为10
Widget w2();   //most vexing parse, 解释为函数声明！
Widget w3{};   //调用Widget的无参构造函数
```
但是统一初始化也有其自身的缺点，因为{}本质上是std::initializer_list，在某些场合会产生让人预想不到的结果，比如Item2中介绍的模板类型推导中关于std::initializer_list的问题。        
另外，如果定义了接受std::initializer_list类型的构造函数，编译器在做重载决议时会尽量的将{}决议到那个构造函数上，即使有其他看起来类型更为匹配的构造函数。
```cpp
class Widget {
public:
  Widget(int i, bool b);
  Widget(int i, double d);
  Widget(std::initializer_list<long double> il); //新增
}

Widget w1(10, true); //调用第一个构造函数
Widget w2{10, true}; //调用第三个构造函数，10和true被转换为long double
Widget w3(10, 5.0);  //调用第二个构造函数
Widget w4{10, 5.0};  //调用第三个构造函数，10和5.0被转换为long double

```
更令人吃惊的是看似为拷贝和移动构造的初始化，也会被那个std::initializer_list版本的构造函数给“劫持”。

但是如果没有到std::initializer<T>中类型T的转换，则会被重载为正常的版本，
```cpp
class Widget {
   public:
     Widget(int i, bool b);               // as before
     Widget(int i, double d);             // as before
     Widget(std::initializer_list<std::string> il);
}

Widget w1(10, true); //调用第一个构造函数
Widget w2{10, true}; //调用第一个构造函数，10和true无法转换为std::string
Widget w3(10, 5.0);  //调用第二个构造函数
Widget w4{10, 5.0};  //调用第二个构造函数，10和5.0无法转换为std::string
```

构造函数没有参数的情况下， 
```cpp
class Widget 
{
public:
  Widget();
  Widget(std::initializer_list<int> il);
  ... 
};
Widget w1;     //无参构造函数
Widget w2{};   //无参构造函数
Widget w3();   //most vexing parse
Widget w4({}); //std::initializer_list<int>版本
Widget w5{{}}; //std::initializer_list<int>版本
```

有时，对()和{}初始化的选择会产生截然不同的结果，比如std::vector，
```cpp
std::vector<int> v1(10, 20); // 含有10个元素，值都为20
std::vector<int> v2{10, 20}; // 含有2个元素，值为10，20
```
这是一个典型的设计失误，容易造成用户的误用。类的设计者应当尽量让()与{}初始化表现的一致。这也强烈的暗示我们，没有充分的理由，不要为类添加接受std::initializer_list的构造函数。

作为类的用户，我们应当坚持一种初始化的方法，只在()和{}表现不同的时候选择正确的初始化。而作为类的作者，特别是需要在泛型类中初始化T的局部变量的时候，没有绝对正确的做法，因为没有办法知道T的具体类型。

### 总结
- 统一初始化是适用范围最广的初始化语法，它能够避免类型截断和most vertex parse
- 在构造函数的重载决议时，{}会匹配到std::initializer_list参数，即使有其它看起来更为匹配的构造函数
- 在模板类中选择一种初始化方法不是一件简单的事



