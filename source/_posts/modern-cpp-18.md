---
title: 读薄Effective Modern C++ (条款18 用std::unique_ptr管理独占所有权的资源)
date: 2019-01-16 02:19:57
tags:
- [C++]
- [Smart pointer]
categories:
- [PL, Tips]
- [Memo]
---

## 条款18 用std::unique_ptr管理独占所有权的资源
指针是强大的，然而原生指针也有着一些的缺点。     

- 难以知道一个原生指针指向的是单个对象还是一个对象数组，所以也难以正确选择delete还是delete[]来释放内存。
- 原生指针无法表达所有权（你是否应该负责释放该指针指向的对象）
- 难以确保所有路径上delete都被正确调用，也就难以确保异常安全
- 无法知道指针是否已经空悬

由于C++缺乏垃圾回收机制，用原生指针来管理内存又是如此的容易出错，C++11添加了一系列的智能指针(std::unique_ptr, std::shared_prt, std::weak_ptr)，用来解决原生指针的种种问题，简化内存管理。

std::unique_ptr用来表示独占所有权语义。所谓独占，意味着一个非空的std::unique_ptr会一直拥有它指向的对象。你无法拷贝一个std::unique_ptr, 而只能移动它。当移动一个std::unique_ptr，所有权会从源指针转移到目标指针（源指针的内容浅拷贝到目标指针，之后源指针置为null）。     
<!-- more -->

std::unique_ptr的一个常见使用是作为工厂函数的返回类型。      
```cpp
template <typename... Ts>        // 返回一个由给定参数
std::unique_ptr<Investment>     // 创建的对象的指针
makeInvestment(Ts&&... params); 
```
调用者可以在一个局部作用域使用返回的std::unique_ptr,
```cpp
{                  
    ...
    auto pInvestment =                      // pInvestment的类型是
          makeInvestment(*arguments*);       // std::unique_ptr<Investment>
    ...
}    // 销毁 *pInvestment
```
std::unique_ptr用于移动语境也是没有问题的，比如用一个容器来接受工厂返回的std::unique_ptr,此时资源的所有权会最终转移到对象内部，最终随着对象声明周期结束而被释放。      

默认情况下，资源销毁是通过对std::unique_ptr内的原生指针使用delete来完成的。也能够自定义删除器(deleter),        

```cpp
template <typename... Ts>
auto makeInvestment(Ts&&... params)   // C++14
{
   auto delInvmt = [](Investment* pInvestment) // 自定义删除器
                   {                           // 内置于makeInvestment
                       makeLogEntry(pInvestment);
                       delete pInvestment;
                   };

    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt); 
    if (...)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if (...)
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if (...)
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```

- delInvmt是从makeInvestment工厂函数返回对象的自定义删除器。所有的自定义删除器都必须有一个指向需要销毁的对象的原生指针作为参数，然后它做的事情是销毁对象的必要工作。在这个例子中，删除器的行为是调用makelogEntry，然后在进行delete。
- 当我们使用自定义的删除器的时候，它的类型要作为std::unique_ptr的第二个模板参数。这也是为什么pInv的类型是std::unique_ptr<Investment, decltype(delInvmt)>。
- 试图将原生指针赋值给std::unique_ptr无法通过编译，因为C++11禁止从原生指针到智能指针的隐式转换。那就是为什么reset被用来让pInvmt得到new出来的对象的所有权。
- 当你使用默认删除器时（即delete），std::unique_ptr对象的大小和原生指针一样。如果使用自动定义删除器，就不是这样了。如果删除器是函数指针，它通常会让std::unique_ptr的大小增加一到两个字(word)。如果删除器是函数对象，std::unique_ptr的大小改变取决于函数对象存储了多少状态。无状态的函数对象不会受到一丝代价，这意味着当自定义删除器可用函数实现也可用不捕获变量的lambda表达式实现时，lambda实现会更好。

std::unique_ptr的另一个常用的地方是在于实现Pimpl Idiom。条款22会专门介绍。      

在C++11中，std::unique_ptr是表达独占所有权的方式，但它最吸引人的一个特性是它能即简单又高效地转化为std::shared_ptr
```cpp
std::shared_ptr<Investment> sp =    // 把 std::unique_ptr转换为
    makeInvestment(argument);       // std::shared_ptr
```
这就是为什么std::unique_ptr如此适合做工厂函数的关键原因，工厂函数不会知道独占所有权语义和共享所有权语义哪个更适合调用者。通过返回一个std::unique_ptr，工厂提供给调用者的是最高效的智能指针，但它不妨碍调用者用std::shared_ptr来替换它。

## 总结

- std::unique_ptr是一个轻量并且只可移动的智能指针，它表达资源的独占权。
- 默认情况下，std::unique_ptr通过delete来销毁资源，但可以指定自定义删除器。有状态的删除器和函数指针作为std::unique_ptr的删除器会增加std::unique_ptr对象的大小。
- 将std::unique_ptr转换为std::shared_ptr是容易的。






