---
title: 读薄Effective Modern C++（条款30 熟悉完美转发失败的情况)
date: 2019-06-05 23:43:45
tags:
- [C++]
- [Universal reference]
- [Rvalue reference]
categories:
- [PL, Tips]
- [Memo]
---

## 条款30 熟悉完美转发失败的情况 

完美转发在大多数时候如期工作，
```cpp
template<typename... Ts>
void fwd(T&& ...params)   //  接受一些参数
{
    f(std::forward<Ts>(param)...);    // 把它们转发到f
}
```
但是也有失败的情形（通过完美转发函数将参数转发给函数f调用与直接将参数传递给f调用的行为不一致。）

<!--more-->

```cpp
f(expression);       // 如果f做某件事
fwd(expression);     // 而fwd里的f做的是不同的事，那么完美转发expression失败
```
有几种实参的类型会导致完美转发失败，

### std::initial_list

```cpp
void f(const std::vector<int>& v);
f({1, 2, 3});   // 正确，“{1，2，3}”被隐式转换为std::vector<int>
fwd({1,2,3})；  // 错误，不能编译
```

正如条款2提到的，这里的问题在于f的参数类型不是`std::initial_list<T>`, 编译器禁止从表达式{1,2,3}推断出一个类型（`std::initial_list<int>`到`std::vector<int>`还需要一次隐式转换，直接比对`std::initial_list<T>`与`std::vector<int>`, T无法被推导为int）。
条款2中同样提到了一个简单的应对办法，用auto声明一个局部变量接受初始化列表，然后再把该局部变量传递给进行转发的函数：

```cpp
auto il = {1, 2, 3}; // il的类型被推导为std::initializer_list<int>
fwd(il);             // 正确，把il转发给f
```

### 0和NULL作为空指针
条款8解释过，当把0和NULL作为空指针传递给一个模板，编译器通常会把传入的参数推断为整型数类型，而绝不会是指针类型。这就导致了0和NULL都不可以作为空指针被完美转发。解决办法也很单纯：用nullptr代替0或NULL。

### 只声明的static const整数成员变量
```cpp
class Widget {
public:
    static const std::size_t MinVals = 28;         // MinVals的声明
    ...
};
...                 // 没有定义MinVals

std::vector<int> weightData;
widgetData.reserve(Widget::MinVals);    // 使用MinVals
```

C++允许对static const的整数变量在声明时提供初始值，而不用进行定义。此时编译器会为这些成员变量的值进行const propagation（常数传播，编译时会直接用常量值替换掉变量），而不需要为这些变量提供内存。
```cpp
f(Widget::MinVals); // 没问题，会被替换为f(28)
```
而通过完美转发调用则可能会出现链接错误，
```cpp
fwd(Widget::MinVals); // 链接错误！
```
这时尽管看起来没有取MinVals的地址，但fwd的参数是个通用引用，然后引用在程序的二进制代码中指针和引用在本质上是同一个东西，引用只是会自动解引用的指针。MinVals相当于通过指针被传递，而传递指针则必须有指针可以指向的内存。以引用传递static const成员变量通常要求它们被定义过，而这个要求会导致未进行定义的代码完美转发失败。

为了让代码可移植，应当为static const成员变量提供一份定义。对于MinVals，代码是这样的：

```cpp
const std::size_t Widget::MinVals; // 在cpp中，不能重复提供初始值
```

### 重载函数名字和模板名字
```cpp
void f(int (*pf)(int));  // pf = "进行处理的函数"
void f(int pf(int));    // 或者可以声明为

int processVal(int value);
int processVal(int value, int priority);

```

```cpp
f(processVal);         // 两种声明都能正确调用
fwd(processVal);      //  第二个版本错误！无法知道调用哪个processVal
```

f需要的是一个指向函数的指针作为它的参数，但是rocessVal既不是个函数指针，也不是一个函数，它是两个不同函数的名字。不过，编译器知道f需要的是接受一个int参数的函数。因此，编译器会选择第一个版本的processVal，然后把那个函数地址传给f。
然而作为函数名称的processVal本身不包含类型信息。没有类型信息，就无法进行类型推导。因此完美转发失败了。
用一个模板函数作为参数传入也会出现同样的问题，因为无法知道使用哪个类型来实例化。

```cpp
template<typename T>
T workOnVal(T param)        // 一个处理值的模板
{ ... }

fwd(workOnVal);        // 错误！哪个workOnVal实例化？
```

像一种解决办法是手动指定想要转发的那个重载或者实例化。

```cpp
using ProcessFuncType = int (*)(int);
ProcessFuncType processValPtr = processVal;     // 指定了processVal的签名
fwd(processValue);                              // 正确
fwd(static_cast<ProcessFuncType>(workOnVal));   // 也是正确的
```
当然，这需要你知道fwd转发的目的函数需要的函数指针类型。


### Bitfields（位域）
最后一种完美转发失败的情况是，当位域被用作函数实参。例如，有一个使用位域定义的IPV4头部：
```cpp
struct IPv4Header {
    std::uint32_t version : 4,
                         IHL : 4,
                         DSCP : 6,
                         ECN : 2,
                         totalLength : 16;
    ...
};
```

```cpp
void f(std::size_t sz);   // 被调用的函数

IPv4Header h;
...
f(h.totalLength);        // 正确
fwd(t.totalLength);      // 错误
```
位域可能是包括字的任意部分（例如，32位int的第3到5个位。），但是没有方法直接获取它们的地址。就像没有办法创建指向任意位的指针（C++表明可指向的最小的东西是一个char），也没有办法对任意位进行绑定引用。可以传递位域的参数种类只有传值和const引用参数。
要解决这个问题，可以先对位域的值进行拷贝，然后对拷贝进行转发。例如，在IPv4Header这个例子，

```cpp
//  拷贝位域的值
auto length = static_cast<std::uint16_t>(h.totalLength);
fwd(length);   // 转发拷贝
```

## 总结
- 当模板类型推断失败或推断出错误的类型时，完美转发会失败。
- 导致完美转发失败的几种实参有：大括号初始化，0和NULL代表的空指针，只有声明的static const整数成员变量，模板函数名字和重载函数名字，位域。
