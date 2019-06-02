---
title: 读薄Effective Modern C++（条款27 熟悉替代重载通用引用的方法)
date: 2019-06-02 15:40:28
tags:
- [C++]
- [Universal reference]
- [Rvalue reference]
categories:
- [PL, Tips]
- [Memo]
---

## 条款27 熟悉替代重载通用引用的方法

条款26解释了对通用引用进行重载会导致很多问题，不过也展示了这种重载有用的地方，该条款探索如何获得我们期望的行为，包括通过设计避免对通用引用进行重载，以及使用通用引用重载时约束它们能够匹配的参数类型。

<!--more-->

几个可能的选项是:

1. 放弃重载
这个方案不太可行，因为构造函数的名称是固定的，另外我们也不愿意放弃重载带来的便利。

2. 通过const T&传递参数
放弃一些效率来保持代码的简单也是值得考虑的。

3. 通过值语义传递参数
往往直接传值高性能却不用增加复杂性的方法是，用值参数替换引用参数。这种设计遵顼条款41的建议，当你知道参数会被拷贝的时候，考虑直接以值传递对象。
```cpp
class Person {
public:
    explicit Person(std::string n)  // 替换 T&& 构造函数
    : name(std::move(n)) {}    // 使用std::move

    explicit Person(int idx)        // 和以前一样
    : name(nameFromIdx(idx)) {}

    ...

private:
    std::string name;
};
```


4. 使用tag dispatch
tag dispatch是泛型代码写作中的常用技术，它在额外模板类型变量，用于控制重载。
tag dispatch的重点是存在一个使用通用引用作为参数但是未重载的函数作为用户的API，这个函数把工作调度到不同的实现函数。
```cpp
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(
      std::forward<T>(name),
      std::is_integral<typename std::remove_reference<T>::type>()
    );
}

template<typename T>
void logAndAddImpl(T&& name, std::false_type)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string nameFromIndx(int idx);     // 和条款26一样

void logAndAddImpl(int idx, std::true_type)
{
    logAndAdd(nameFromIdx(idx);
}
```


5. 约束接受通用引用的模板
tag dispatch并不能避免编译器生成的拷贝构造及移动构造被劫持的问题。这里我们需要使用std::enable_if来限制模板的实例化，std::enable_if的基础是SAFINE。
```cpp
class Person {
public:
    template<
      typename T,
      typename = std::enable_if_t<
        !std::is_base_of<Person, std::decay_t<T>>::value
        &&
        !std::is_integral<std::remove_reference_t<T>>::value
      >
    >
    explicit Person(T&& n)       // 接受std::string类型
    : name(std::forward<T>(n))   // 和 可以转换为string的参数的构造函数
    { ... }

    explicit Person(int idx)     // 接受整型参数的构造函数
    : name(nameFromIdx(idx))
    { ... }

    ...    // 拷贝和移动构造， 等等

private:
    std::string name;
};
```

## 总结
- 替代通用引用和重载的方法包括使用有区别的函数名（不重载）、以lvalue-reference-to-const形式传递参数、以值语义传递参数和使用tag dispatch。
- 借助std::enable_if限制模板通用引用和重载能够同时使用。
- 通用引用参数一般有性能优势，但它们更加难以使用。


