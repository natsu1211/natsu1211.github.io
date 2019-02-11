---
title: 读薄Effective Modern C++ (条款19 用std::shared_ptr管理共享所有权的资源)
date: 2019-01-27 01:03:55
tags:
- [C++]
- [Smart pointer]
categories:
- [PL, Tips]
- [Memo]
---
## 条款19 用std::shared_ptr管理共享所有权的资源

正如其名, std::shared_ptr用于表达共享所有权语义，也就是说，可以有多个std::shared_ptr指向同一个对象，并且保证在没有std::shared_ptr指向该对象时，对象被正确的释放。      

那么如何知道不再有任何std::shared_ptr指向某个对象/资源呢？答案是引用计数。std::shared_ptr内部除了包含原生指针，还包含一个控制块（的指针），该控制块（一个struct）里有一个整数字段，用于记录有多少std::shared同时指向该对象。      
std::shared_ptr的非移动构造函数会增加这个计数，拷贝和赋值运算也会。（如果shared_ptr对象sp1和sp2指向不同的对象，那么赋值语句“sp1 = sp2”会导致sp1指向sp2指向的对象。赋值语句还会导致sp1原来指向的对象的引用计数减1，sp2指向的对象的引用计数加1。）如果一个std::shared_ptr在操作引用计数后发现引用计数为0，意味着没有其他的std::shared_ptr指向该资源，便释放资源。
<!-- more -->

上面的说明包含了以下信息：      

- std::shared_ptr的大小通常是原生指针的两倍，因为它包含一个指向资源的原生指针，还有资源的引用计数。引用计数所用的内存一定是动态分配的。因为，但是指向的对象对引用计数一无所知，它们也没有存储引用计数的地方。
- 增加和减少引用计数一定是原子操作，因为在不同的线程中会同时存在读和写。例如，在一个线程中，std::shared_ptr正在析构（会减少引用计数），同时在另一个线程，指向相同资源的std::shared_ptr正在被拷贝（会增加引用计数）。原子操作通常会比非原子操作慢，所以尽管引用计数通常只占据一个word的大小，也应该认为读写引用计数是比较昂贵的。

类似于std::unique_ptr，std::shared_ptr使用delete作为默认的释放资源手段，但它也支持自定义删除器。但是，不同于std::unique_ptr，自定义删除器的类型不是std::shared_ptr类型的一部分，

```cpp
auto loggingDel = [](Widget *pw)
                  {
                      makeLogEntry(pw);
                      delete pw;
                  };

std::unique_ptr<Widget, decltype(loggingDel)>            // 删除器的类型是
                     upw(new Widget, loggingDel);        // 指针类型的一部分

std::shared_ptr<Widget>              // 删除器的类型不是指针类型的一部分
            spw(new Wiget, loggingDel); 
```
这意味着持有不同类型自定义删除器的std::shared_ptr也可以放进同一个容器里。
```cpp
auto customDelete1 = [](Widget *pw) { ... };   // 自定义删除器
auto customDelete2 = [](Widget *pw) { ... };   // 两个类型不同
std::shared_ptr<Widget> pw1(new Widget, customDelete1);
std::shared_ptr<Widget> pw2(new Widget, customDelete2);

std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```
还有一个和std::unique_ptr不同，指定自定义删除器不会改变std::shared_ptr对象的大小。不管使用怎样的删除器，一个std::shared_ptr对象的大小都是两个原生指针。它的内存模型如下图所示，
![](https://raw.githubusercontent.com/natsu1211/pics/master/shard_ptr.jpg)

不是类型的一部分的原因也在这，因为在控制块是保存在堆上的，std::shared_ptr只是持有该控制块的指针。

通常情况下，创建std::shared_ptr的函数不可能知道是否已经有其它的std::shared_ptr指向该对象，所以控制块创建要服从以下规则：

- std::make_shared（条款21）总是会创建控制块。它是加工一个刚new出来的对象，所以这个对象一定不会有控制块。
- 当std::shared_ptr由独占所有权指针构造时，因为独占所有权的指针不会使用控制块，控制块需要被创建。
- 当以原生指针为参数调用std::shared_ptr的构造函数时，也会创建控制块。

这样的规则会导致一个问题：由单一的原生指针构造std::shared_ptr构造多次会导致未定义行为，因为指向的对象会有多个控制块。多个控制块意味着多个引用计数，多个引用计数意味着多次被销毁。所以，

- 构造std::shared_ptr时应当尽量使用std::make_shared
- 需要自定义删除器而无法使用std::make_shared时,也应当在初始化shared_ptr时直接使用new出来的对象，如`std::shared_ptr<Widget> spw1(new Widget， loggingDel); `，这样可以避免原生指针被多次初始化。

std::shared_ptr涉及到动态分配的控制块、任意大的删除器和删除器的内存分配、虚函数的开销、但鉴于它提供的功能，std::shared_ptr要求的开销是情有可原的。典型情况下，std::shared_ptr是通过std::make_shared创建的，使用的是默认的删除器和默认的分配器，控制块的大小只有两到三个字（word），然后动态分配的开销基本没有（具体细节看条款21）。解引用一个std::shared_ptr的开销不比解引用原生指针大，操作引用计数涉及到一或两个原子操作，这些操作通常会转化为一条机器指令，所以尽管它们可能比非原子指令开销大，但是它们依旧是单一机器指令。通常情况下，管理对象的std::shared_ptr只用一次控制块里的虚函数操作，那便是当对象被销毁时。

作为这些适量开销的交换，我们得到了动态分配资源的自动生命期管理。如果共享所有权不是必须的，那么std::unique_ptr为更好的选择。

## 总结

- std::shared_ptr提供了一种方便的管理共享生命周期的资源的方法。
- 与std::unique_ptr相比，std::shared_ptr对象通常是它的两倍大，需要控制块和原子操作的引用计数。
- 默认销毁资源的方式是delete，但支持自定义删除器。删除器的类型不影响std::shared_ptr的类型。
- 避免用原生指针变量来创建std::shared_ptr。