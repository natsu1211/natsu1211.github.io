---
title: 读薄Effective Modern C++（条款26 避免对通用引用进行重载)
date: 2019-06-01 23:36:29
tags:
- [C++]
- [Universal reference]
- [Rvalue reference]
categories:
- [PL, Tips]
- [Memo]
---
## 读薄Effective Modern C++（条款26 避免对通用引用进行重载)

接受通用引用作为参数的函数是C++最贪婪的函数，它们几乎接收所有类型的参数，从而创建的精确匹配，使得重载决议时用户希望调用的函数无法被匹配到，例如：
```cpp
template<class T>
void logAndAdd(T&& name)
{
    // do log
}

void logAndAdd(int index)
{
    std::string name = nameFromIndex(index);
    logAndAdd(name);
}

std::string petName("Darla");

logAndAdd(petName); //这三个都是使用T&&的重载
logAndAdd(std::string("Persephone"));
logAndAdd("PattyDog");

logAndAdd(22);    // 使用int重载

short nameIndex;
...
logAndAdd(nameIndex); // 编译错误，因为重载为T&&版本，std::string没有一个接受short类型的构造函数
```

这就是为什么结合重载和通用引用是个糟糕的想法。

而更糟糕的是使用完美转发的构造函数，

```cpp
class Person {
public:
    template<typename T>
    explicit Person(T&& n)         // 完美转发构造函数
    : name(std::forward<T>(n)) {}  // 初始化成员变量

    explicit Person(int idx)   // 接受int的构造
    ：name(nameFromIdx(idx)) {}
    ...
private:
    std::string name;
};
```
```cpp
Person p("Nancy");
auto cloneOfP(p);   // 从p创建一个新Person，这不能通过编译
```

cloneOfP被一个非const左值p来进行初始化，实例化之后，Person的代码变成这样：

```cpp
class Person {
public:
    explicit Person(Person& n)      // 从完美转发模板构造实例化而来
    : name(std::forward<Person&>(n)) {} 

    explicit Person(int idx);

    Person(const Person& rhs); 

    ...
};
```

调用拷贝构造需要对p添加const才能匹配参数类型，但是调用实例化的模板构造函数却是精确匹配。编译器做了它们应该做的事情：调用更加匹配的函数。因此，“拷贝”一个非const的左值Person，会被完美转发构造函数处理，而不是拷贝构造函数。

如果我们稍稍改一下代码，让对象拷贝const对象，我们就会得到完全不一样结果：
```cpp
const Person cp("Nancy");  // 对象是const的
auto cloneOfP(cp);      // 调用拷贝构造
```

因为对象拷贝的对象是const的，它会精确匹配拷贝构造函数。模板化构造函数可以实例化出一样的签名，
```cpp
class Person {
public:
    explicit Person(const Person& n);    // 实例化的模板构造函数

    Person(const Person& rhs);   // 编译器生成的拷贝构造函数

    ...
};
```
根据C++的重载决策规则，当一个模板实例化函数和一个普通函数匹配度一样时，优先使用普通函数。

如果考虑继承，同样会有类似的问题，
```cpp
class SpecialPerson :  public Person {
public:
    SpecialPerson(const SpecialPerson& rhs)  // 拷贝构造函数
    : Person(rhs)              // 调用基类的完美转发构造函数
    { ... }

    SpecialPerson(SpecialPerson&& rhs)    // 移动构造函数
    : Person(std::move(rhs))   // 编译错误！调用基类的完美转发构造函数！
    { ... }
};
```

## 总结：

- 对通用引用进行重载几乎总是会导致通用引用版本被调用。
- 完美转发构造函数应当尽量避免，因为在接受非const左值作为参数时，它们通常比拷贝构造匹配度高，它们还能劫持派生类调用的基类的拷贝及移动构造。
