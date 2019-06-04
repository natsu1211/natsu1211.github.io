---
title: 读薄Effective Modern C++（条款28 理解引用折叠)
date: 2019-06-03 01:49:30
tags:
- [C++]
- [Universal reference]
- [Rvalue reference]
categories:
- [PL, Tips]
- [Memo]
---

## 条款28 理解引用折叠

尽管C++禁止声明引用的引用，但编译器在模板实例化的过程中可以产生它们。当编译器生成对引用的引用时，随后会发生引用折叠。

引用折叠按照以下规则工作， 
```
如果两个引用中有一个是左值引用，那么折叠的结果是一个左值引用。否则（即两个都是右值引用），折叠的结果是一个右值引用。
```

<!--more-->

引用折叠也是std::forward能够工作的关键，     
以下是std::forward的略微简略的实现，

```cpp
template<typename T>          // 在命名空间std中
T&& forward(typename remove_reference<T>::type& param)
{
    return static_cast<T&&>(param);
}
```
它的关键就在于当传给函数f的是个左值的Widget的时候，T会被推断为Widget&，然后调用std::forward会让它实例化为std::forward<Widget&>,
```cpp
Widget& && forward(Widget& param)
{    
    return static_cast<Widget& &&>(param);    
}
// 发生引用折叠
Widget& forward(Widget& param)
{    
    return static_cast<Widget&>(param);    
}
```
正如以上，当传递左值实参给std::forward，会得到一个左值引用。左值引用也是左值，所以传递一个左值给std::forward，会导致std::forward返回一个左值。

如果传递的是右值的Widget。在这种情况下，函数f的类型参数T会被推断为Widget，
```cpp
Widget&& forward(Widget& param)
{    
    return static_cast<Widget&&>(param);    
}
```
这里没有对引用的引用，所以没有进行引用折叠。所以这也就是最终的实例化结果。返回的是右值引用。

引用折叠出现在四种上下文。     
1. 模板实例化。       
2. auto变量的类型推导。这本质上和模板实例化相同，因为auto变量的类型推导和模板类型推导本质上相同（条款2）。      
3. 使用typedef或using类型别名声明（条款9）。如果在typedef创建或者评估期间，出现了引用的引用，会发生引用折叠。      
4. 使用decltype时。如果在分析decltype的类型期间，出现了引用的引用，会发生引用折叠。（decltype详见条款3)

简而言之，引用折叠会发生在需要类型推导的时候。

## 总结

- 引用折叠会出现在4中上下文：模板实例化，auto类型生成，typedef和类型别名声明的创建和使用，decltype。
- 当编译器在一个引用折叠上下文中生成了对于引用的引用时，结果会变折叠成单一引用。如果原来的引用中只要有一个是左值引用，结果就是左值引用。否则，结果是右值引用。
- 通用引用本质是出现在引用折叠的上下文中的右值引用。

