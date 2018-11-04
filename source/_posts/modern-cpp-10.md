---
title: 读薄Effective Modern C++ (Item10 优先使用有作用域的enum而不是无作用域的enum)
date: 2018-10-28 02:18:56
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---

## Item10 优先使用有作用域的enmus而不是无作用域的enum
一般而言，{}会建立一个新的作用域，将作用域内部和外部的名称隔离开。但是这对于C++98风格的enum中的枚举元素并不成立。枚举元素和包含它的枚举类型同属一个作用域，这意味着在这个作用域中不能再有同样名称的定义：
<!-- more -->
```cpp
enum Color { black, white, red};                 // black, white, red 和
	                                             // Color  同属一个作用域

auto white = false;                              // 错误！因为 white
	                                             // 在这个作用域已经被声明过
```
而enum class则不会有该问题，
```cpp
enum class Color { black, white, red }; // black, white, red
auto white = false;                     // fine
Color c = white;                        // error!
Color c = Color::white;                 // fine
auto c = Color::white;                  // fine 
```

上面的理由已经足够让我们选择enum class，但是enum class还有一个优势，它的枚举元素是强类型的。
对于无作用域的enum，枚举元素可以隐式转换为数值类型，

```cpp
enum Color { black, white, red };
std::vector<std::size_t> primeFactors(std::size_t x);
Color c = red;
...
if (c < 14.5) {                     //能通过编译！
  auto factors = primeFactors(c);   //能通过编译！
}
```
一个表示颜色的变量能够和double类型作比较，以及能用隐式转换为std::size_t类型，显然不是我们所期望的，使用enum class的话就能达到我们期望的效果，
```cpp
enum class Color { black, white, red };
std::vector<std::size_t> primeFactors(std::size_t x);
Color c = Color::red;
...
if (c < 14.5) {                     //编译错误！
  auto factors = primeFactors(c);   //编译错误！
}
```
如果确实需要进行类型转换，可以用static_cast进行显式转换。

有些时候，无作用域的enum的枚举项能够隐式转换为整数的特性会比较方便使用，比如作为有意义的索引值，
```cpp
using UserInfo = std::tuple<std::string,
                           std::string,        // email
                           std::size_t> ;      // reputation
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
...
auto val1 = std::get<1>(uInfo);       // 不清楚索引1对应的数据是什么
auto val2 = std::get<uiEmail>(uInfo); // 隐式转换为1，清楚的知道是获取email字段
```
而enum class因为无法隐式转换，需要写成，
```cpp
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);

```
但是这种写法的可读性不是太好，我们可以提供一个转换函数来增加可读性，
```cpp
template<typename E>    // C++14 constexpr auto
toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
}
```
```cpp
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

## 总结
- C++98风格的enum是没有作用域的enum
- 有作用域的枚举体的枚举元素仅仅对枚举体内部可见。只能通过类型转换（cast）转换为其他类型
- 有作用域和没有作用域的enum都支持指定潜在类型。有作用域的enum的默认潜在类型是int。没有作用域的enum没有默认的潜在类型。
- 有作用域的enum总是可以前置声明的。没有作用域的enum只有当指定潜在类型时才可以前置声明。


