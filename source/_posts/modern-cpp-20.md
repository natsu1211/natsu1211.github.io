---
title: 读薄Effective Modern C++ (条款20 用std::shared_ptr管理共享所有权的资源)
date: 2019-02-02 17:55:24
tags:
- [C++]
- [Smart pointer]
categories:
- [PL, Tips]
- [Memo]
---

## 条款20 把std::weak_ptr当作类似std::shared_ptr的可空悬指针使用

std::weak_ptr是用于表示弱引用的智能指针。std::weak_ptr不是一个单独使用的智能指针，它要和std::shared_ptr搭配使用。std::weak_ptr不能被解引用，也不能检测是否为空。

std::weak_ptr通常是由std::shared_ptr创建而来，
```cpp
auto spw = std::make_shared<Widget>();  // spw是std::shared_ptr<Widget>
                                        // 引用计数为1
...
std::weak_ptr<Widget> wpw(spw);         // wpw指向spw指向的Widget，引用计数仍然为1
...
spw = nullptr;                          // 引用计数变成0，Widget被销毁，wpw现在变成空悬指针
```
空悬的std::weak_ptr被称作是已经过期（expired）的，你可以直接检查它，     
```cpp
if (wpw.expired()) ... // 如果wpw指向的不是一个对象
```
<!-- more -->

不过一般状况是，你去检查std::weak_ptr是否过期，如果没有过期（即不是空悬），就要取得它指向的对象。但是这件事要比想象的困难，因为std::weak_ptr没有解引用操作，所以没有办法写出解引用的代码。就算有这个操作，单独的检查操作和解引用操作会引出一个竞争条件：在调用检查操作和解引用操作之间，另一个线程可能正在重赋值或销毁最后一个指向对象的std::shared_ptr，因此导致对象被销毁，这样解引用会产生未定义行为。

我们需要的是一个原子操作，检查std::shared_ptr是否过期，没有的话，获得它指向的对象。
你以通过lock函数或是接受std::weak_ptr为参数的std::shared_ptr构造函数来做这件事。
```cpp
std::shared_ptr<Widget> spw1 = wpw.lock();  // 如果wpw过期了，spw1为空
auto spw2 = wpw.lock();                     // 效果和上面相同， 不过使用了auto
```
```cpp
std::shared_ptr<Widget> spw3(wpw);          // 如果wpw过期，抛出std::bad_weak_ptr
```

std::weak_ptr的用途主要有，

- 用于实现观察者模式。在多数实现中，每个Subject（主题）包含指向Observer(观察者)指针作为成员变量，这让主题在状态改变时很容易通知观察者。主题没有兴趣控制观察者的生命期，但是应当确保如果观察者被销毁，随后主题不会试图使用它。一个合理的设计是让每个主题持有一个元素为std::weak_ptr的容器，std::weak_ptr指向主题的每个观察者，因此主题在使用观察者之前可以查看其是否已经被销毁。
- 用于避免循环引用。不过，值得注意的是需要用std::weak_ptr回避循环引用的情况并不常见。在严格分层的数据结构，例如树中，子结点通常只属于父结点，当父结点被销毁，它的子结点也会被销毁。因此通常情况下，父结点指向子结点的指针最好是std::unique_ptr，而子结点指向父结点的指针可以用原生指针安全实现，因为子结点的生命期决不会比父结点长，所以不必担心原生指针会空悬。

最后，从效率的角度来看，std::weak_ptr的开销和std::shared_ptr差不多。std::weak_ptr对象的大小与std::shared_ptr对象相同，它们使用者与shared_ptr的一样的控制块（看条款19），也带有涉及引用计数操作的构造函数、析构函数、赋值操作。std::weak_ptr管理的是弱引用计数，详细细节看条款21。

## 总结
- 把std::weak_ptr当作类似std::shared_ptr的、可空悬的指针使用
- std::weak_ptr的潜在用途包含缓存，观察者链表，防止std::shared_ptr循环