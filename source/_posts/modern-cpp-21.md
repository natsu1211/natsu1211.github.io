---
title: 读薄Effective Modern C++ (条款21 比起new，更倾向于使用std::make_unique和std::make_shared)
date: 2019-02-11 13:56:25
tags:
- [C++]
- [Smart pointer]
categories:
- [PL, Tips]
- [Memo]
---

## 条款21 比起new，更倾向于使用std::make_unique和std::make_shared

std中有三个make函数（std::make_unique, std::make_shared, std::allocate_shared）来帮助我们创建智能指针。它们把任意集合的参数完美转发给动态分配对象的构造函数，然后返回一个指向那对象的智能指针。std::allocate_shared的行为与std::make_shared类似，除了它第一个参数是个分配器，指定动态分配对象的方式。

对比直接通过new来创建智能指针，make系列函数来创建主要有三点优势。

- 更简洁的代码
- 能够保证异常安全
- 更高的效率

<!-- more -->

首先，直接使用new来创建智能指针时需要重复写下对象的类型名称，而make函数则只需要写一次。
```cpp
auto upw1(std::make_unique<Widget>());  // 使用make函数

std::unique_ptr<Widget> upw2(new Widget);  // 不使用make函数

auto spw1(std::make_shared<Widget>());  // 使用make函数

std::shared_ptr<Widget> spw2(new Widget);  // 不使用make函数
```
当然这不足以成为决定性的理由。更加重要的是make系列函数能够保证异常安全，
考虑以下代码，假如我们有个函数，根据优先级来处理Widget：     

```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority);
```
另外我们有个计算优先级的函数，     

```cpp
int computePriority();
```
然后我们用它和new创建的智能指针作为参数调用processWidget：     

```cpp
processWidget(std::shared_ptr<Widget>(new Widget), 
              computePriority());    // 可能会资源泄漏
```
就如注释所说，这代码中new出来的Widget可能会泄漏，但是为什么？std::shared_ptr是为了防止资源泄漏而设计的，当最后一个指向资源的std::shared_ptr对象消失，它们指向的资源也会被销毁。如果每个人无论什么地方都使用std::shared_ptr，C++还会发生内存泄漏么？

原因在于，C++标准规定函数的参数在函数运行前必须被求值，所以调用processWidget时，在processWidget开始前执行会发生以下事情：        

1. 表达式“new Widget”会被求值，一个Widget对象必须在堆上被创建。
2. std::shared_ptr的接收原生指针的构造函数一定要执行。
3. computePriority一定要执行。       

编译器在生成代码时不会保证上面的执行顺序，“new Widget”一定会在std::shared_ptr构造函数之前执行，因为构造函数需要new的结果，但是computePriority可能在它们之前就被调用了，可能在它们之后，也可能在它们之间。所以，编译器生成代码的执行顺序有可能是这样的：

1. 执行“new Widget”。
2. 执行computePriority。
3. 执行std::shared_ptr的构造函数。

如果生成的代码真的是这样，那么在运行时，如果computePriority产生了异常，步骤1中动态分配的Widget就泄漏了，因为它没有被步骤3中的std::shared_ptr保存。

使用std::make_shared_ptr可以避免这问题。这样调用代码：      

```cpp
processWidget(std::make_shared<Widget>(), computePriority())
```

在运行期间，std::make_shared和computePriority都有可能先被调用，如果先调用的是std::make_shared，那么指向动态分配Widget对象的原生指针会安全地存储在要返回的std::shared_ptr中，然后再调用computePriority。如果computePriority产生异常，std::shared_ptr的析构函数就会销毁持有的Widget。而如果先调用的是computePriority，并且产生异常，std::make_shared就不会被执行，因此没有动态分配的Widget对象需要担心。     

这里的要点在于make函数保证了智能指针的初始化与对象内存的动态分配在同一个表达式中运行。     
另外，以上的说明对std::unique_ptr和std::make_unique同样适用。

最后，使用std::make_shared允许编译器生成更小、更快的代码。      

```cpp
std::shared_ptr<Widget> spw(new Widget); //两次内存分配
auto spw = std::make_shared<Widget>();   //一次内存分配
```
条款19说明了每个std::shared_ptr内都含有一个指向控制块的指针，这控制块的内存是由std::shared_ptr的构造函数分配的，那么直接使用new，需要为Widget分配一次内存，还需要为控制块分配一次内存。而使用std::make_shared的时候，会分配一大块内存来同时持有Widget对象和控制块。这种优化减少了程序的静态尺寸，因为代码只需要调用一次内存分配函数，然后它增加了代码执行的速度。这个优势同样存在于std::allocate_shared。       

但是，make系列函数也有他们自身的限制。            

1. 不可以指定自定义删除器
2. make函数内，完美转发使用的是圆括号。

这两种情况下应当直接使用各个智能指针的构造函数。             

对于1，如果需要使用自定义删除器，又想要防止容易发生的异常安全问题。最好的办法就是确保当你直接使用new时，用一条语句执行——把new的结果马上传递给智能指针的构造函数，并且该语句就做这一件事。

```cpp
void processWidget(std::shared_ptr<Widget> spw, int priority); // 如前
void cusDel(Widget *ptr);      //  自定义删除器

processWidget(                 // 如前，资源可能泄漏
   std::shared_ptr<Widget>(new Widget, cusDel),
   computePriority()
);

std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(spw, computeWidget);  // 正确，但不是最佳，因为spw是个左值，会进行拷贝构造

processWidget(std::move(spw), computePriority());  // 将spw转换为右值，得到与非异常安全版本一样的效率。因为对于std::shared_ptr来说，移动构造不需要像拷贝构造一样进行昂贵的引用计数增加操作。
```

对于2，可以借助auto来进行类型推导，从而创建一个std::initializer_list，

```cpp
// 创建 std::initializer_list
auto initList = {10, 20};

// 使用std::initializer_list创建std::vector，容器中只有两个元素
auto spv = std::make_shared<std::vector<int>>(initList);
```
对于std::make_unique来说，上面就是故事的全部了，但是对于std::make_shared来说，还有一些边缘情况值得知道。    

3. 一些类重定义了自己的new和delete操作符函数，这些函数的出现暗示着常规的全局内存分配和回收不适合这种类型的对象。通常情况下，设计这些函数只有为了精确分配和销毁对象，例如，Widget对象的new和delete操作符是为了精确分配和回收大小为sizeof(Widget)的内存块设计的。这两个函数不适合std::shared_ptr的自定义分配（借助std::allocate_shared）和回收（借助自定义删除器），因为std::allocate_shared请求内存的大小不是对象的尺寸，而是对象尺寸加上控制块尺寸。所以，使用make函数为那些自定义了new和delete操作符的类创建对象通常是个糟糕的想法。
4. 正如之前提到的，std::make_shared所创建的std::shared_ptr的控制块与它管理的对象放在同一块内存。当引用计数为0时，对象被销毁，但是，它使用的内存不会释放，除非控制块也被销毁。但是，只要有std::weak_ptr指向控制块（weak count大于0），控制块就必须继续存在，而只要控制块存在，容纳它的内存快也就不能被释放。也就是说，通过make函数创建对象所分配的内存，要直到最后一个指向它的std::shared_ptr和std::weak_ptr对象销毁，才能被回收。如果对象的类型非常大，并且最后一个std::shared_ptr销毁和最后一个std::weak_ptr销毁之间的时间间隔很大，那么是对象销毁和内存被回收之间的会有延迟，这在某些内存紧张的系统上可能会造成问题。而直接使用new来创建智能指针就不会有这种问题，因为控制块的内存和对象自身的内存是分开的。

## 总结

- 相比于直接使用new，make函数更加简洁，更加异常安全，而且std::make_shared和std::allocate_shared生成的代码更小更快。
- 不适合使用make函数的场合包括需要指定自定义删除器和想要传递大括号初始值。
- 对于std::shared_ptr，使用make函数可能是不明智的额外场合包括
    - 自定义内存管理函数的类
    - 内存紧张的系统中，对象非常大，并且std::weak_ptr生命周期长于std::shared_ptr。




