---
title: 读薄Effective Modern C++ (条款8 优先使用nullptr而不是0和NULL)
date: 2018-10-27 00:35:12
tags:
- [c++]
categories:
- [编程语言]
- [读书笔记]
---

## 条款8 优先使用nullptr而不是0和NULL 
使用0或者NULL来初始化空指针的最大问题是0是int类型，NULL根据定义可能是int或是long，但是不会是个指针类型。编译器只是勉强将0解释为空指针。

nullptr作为C++11中新增的关键字，它的优势是它不再是int类型，而是一个可以表示指向任意类型的指针的类型。nullptr在内部的类型是std::nullptr_t，这是一个模板类，内部定义了转换操作符，可以隐式的转换为所有的原始的指针类型。
<!-- more -->

在函数重载决议时，nullptr可以避免错误的类型重载。
```cpp
void f(int);  // 函数f的三个重载
void f(bool);
void f(void*);

f(0);         // 调用 f(int)，而非f(void*)

f(NULL);      // 可能无法编译，可能调用f(int)
              // 不可能调用 f(void*)
f(nulltpr);   // 调用 f(void*) 
```
另外，当你写出以下代码时，
```cpp
auto result = findRecord( /* arguments */);
if(reuslt == nullptr){
	...
}
```
一眼就能明白result是个指针，这能够增加代码的可读性。

### 总结
- 相较于0和NULL，优先使用nullptr
- 避免整数类型和指针类型之间的重载