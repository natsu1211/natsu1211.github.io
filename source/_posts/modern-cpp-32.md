---
title: 读薄Effective Modern C++（条款32 使用初始化捕获来把对象移动到lambda闭包中）
date: 2019-06-15 18:17:11
tags:
- [C++]
- [Lambda]
categories:
- [PL, Tips]
- [Memo]
---

## 条款32 使用初始化捕获来把对象移动到lambda闭包中
有时候，你想要的既不是值捕获，也不是引用捕获。而是想要把一个只可移动对象（例如，std::unique_ptr或std::future类型对象）放入闭包中，C++11没有办法做这事，但是C++14之后则可以通过init capture或者叫做generalized lambda capture来做到。

```cpp
class Widget {
public:
    ...
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;
private:
    ...
};

auto pw = std::make_unique<Widget>();  //创建Widget

...              // 配置*pw

auto func = [pw = std::move(pw)]  // 以std::move(pw)来初始化闭包中成员变量pw
            { return pw->isValidated() && pw->isArchived(); };
```
<!--more--->

如果配置不是必须的，还可以直接写成，

```cpp
auto func = [pw = std::make_unique<Widget>()]
            { return pw->isValidated() && pw->isArchived(); };
```

init capture的代码部分是`pw = std::move(pw)`，它的意思是：在闭包中创建一个成员变量pw，然后使用局部变量pw来移动初始化构造那个成员变量。

如果你还在使用C++11,可以手动写一个类来实现类似功能，需要记住的是一个lambda表达式，编译器为会其生成一个类，并且会创建那个类的对象。

```cpp
class IsValAndArch {
public:
    using DataType = std::unique_ptr<Widget>;

    explicit IsValAndArch(DataType&& ptr)
    : pw(std::move(ptr)) {}

    bool operator()() const
    { return pw->isValidated() && pw->isArchived; }
private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```
或者使用std::bind来绕开C++11中闭包成员无法移动构造的问题，
```cpp
auto func = 
    std::bind(
      [](std::vector<double>& data) mutable
      { /* uses of data */ },
      std::move(data);
  );
```


## 总结

- 使用C++14的init capture来把对象移到到闭包。
- C++11可以借助手写类或std::bind模仿init capture。

