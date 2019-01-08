---
title: 读薄Effective Modern C++ (条款15 尽可能的使用constexpr)
date: 2019-01-06 21:47:15
tags:
- [c++]
categories:
- [编程语言]
- [读书笔记]
---

## 条款15 尽可能的使用constexpr
constexpr可以说是c++11中最让人困惑的关键字，它用来修饰变量或函数，表明该变量或函数的值能够在编译期获得。这些变量和函数可以用于需要一个常量表达式（constant expression）的地方。例如，
```cpp
int sz;   // 非constexpr变量
...
constexpr auto arraySize1 = sz;   // 错误，编译期间不知道sz的值

std::array<int, sz> data1;        // 错误，同样的问题

constexpr auto arraySize2 = 10;   // 正确，10在编译期间是常量

std::array<int, arraySize2> data2;  // 正确，arraySize2是constexpr的
```
<!-- more -->

实际上，当constexpr用于修饰对象或非静态成员函数时（当传递适当的参数时，下面会说），该对象和函数同时也是const的。      
作为补充，constexpr变量需要满足以下条件，否则会产生编译错误。          

- 其类型必须是字面类型 (LiteralType) 。
- 它必须被立即初始化
- 其初始化的完整表达式，包括所有隐式转换、构造函数调用等，都必须是常量表达式

而当constexpr修饰函数时，情况有所不同，不能保证该函数的值是编译期常量。      

- constexpr函数可以用在需求编译期间常量的上下文中（比如上面创建std::array的表达式中）。在这种上下文中，如果调用时传进去的实参的值在编译期间已知，那么函数的结果会在编译期间计算。如果任何一个参数的值在编译期间未知，代码将不能通过编译。     
- 如果用一个或者多个在编译期间未知的值作为参数调用constexpr函数，函数的行为和普通的函数一样，在运行期间计算结果。         

不准确的来说，constexpr函数需要保证函数的返回值和每个参数都为字面类型（LiteralType），否则会产生编译错误。
(因为条件较多，constexpr函数需要满足的更为具体的规则可以参考[这里](https://en.cppreference.com/w/cpp/language/constexpr)）

字面类型本质上就是一些在编译期间可确定值的类型。在C++14之前，除了void之外的内置类型都是字面类型（C++14开始void也算，因此C++14开始constexpr的返回值类型可以为void，而C++11不行），不过用户定义的类型也有可能是字面值类型，因为我们可以将构造函数和其他成员函数也声明为constexpr的：
```cpp
class Point {
public:
   constexpr Point(double xVal = 0, double yVal = 0) noexcept
   : x(xVal), y(yVal)
   {}

   constexpr double xValue() const noexcept { return xVal; }
   constexpr double yValue() const noexcept { return yVal; }

   constexpr void setX(double newX) noexcept { x = newX; } // c++14
   constexpr void setY(double newY) noexcept { y = newY; } // c++14

private:
   double x, y;
};
```
如果传进来的参数在编译时已知，Point可以用constexpr初始化：
```cpp
constexpr Point p1(9.4, 27.7);  // 正确，在编译时“运行”constexpr构造
constexpr Point p2(28.8, 5.3);  // 也正确
```
（constexpr构造函数需要满足的条件可以参考[这里](https://en.cppreference.com/w/cpp/language/constexpr)，什么样的类型是LiteralType可以参考[这里](https://en.cppreference.com/w/cpp/named_req/LiteralType)）      

同时我们也可以写出操作这些编译期对象的constexpr函数，
```cpp
constexpr Point reflection(const Point &p) noexcept
{
    Point result;     // create non-const Point

    result.setX(-p.xValue());
    result.setY(-p.yValue());

    return result;
}
```

最终用户的代码会像以下这样，
```cpp
constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);

constexpr auto reflectedMid =      // reflectedMid的值是（-19.1, -16.5）
reflection(mid);                   // 而且在编译期间就知道了
```


因为到了C++14/C++17/C++20，constexpr的限制已经放的如此的宽松。这意味着我们有足够的设施能够将运行时的工作转移到编译时，这也代表我们的程序可以运行的更快（当然编译时间会变长）。

通过尽可能地使用constexpr，我们最大化了对象和函数的可能使用的情况（既可以像传统运行期函数一样使用，也可以用在需要常量表达式的上下文中）。注意，constexpr是一个对象或函数接口很重要的一部分。constexpr表明“我可以用于需求常量表达式的上下文”，如果你把对象或者函数声明为constexpr，用户就有可能把它用于这种上下文。后来，如果你觉得你使用constexpr是个错误，然后你删除了它，这样就可能造成用户大量代码无法编译（例如，为了调试添加I/O函数会导致这种问题，因为I/O语句通常不允许出现在constexpr函数）。因此，一旦选择了constexpr作为接口的一部分，就需要长期保证该接口不会改变。

## 总结
- constexpr对象是const的，它需要用编译期间已知的值初始化。
- constexpr函数在传入编译期已知值作为参数时，函数的返回值也能在编译期获得。
- constexpr对象和函数比non-constexpr对象和函数能运用在更加广泛的语境中。
- constexpr是对象和函数接口的一部分。
