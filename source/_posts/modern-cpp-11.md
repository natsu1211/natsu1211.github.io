---
title: 读薄Effective Modern C++ (条款11 优先使用delete关键字删除函数而不是不实现的私有函数)
date: 2018-11-01 00:22:48
tags:
- [C++]
- [delete]
categories:
- [PL, Tips]
- [Memo]
---

## 条款11 优先使用delete关键字删除函数而不是不实现的私有函数 
有时我们会希望能够禁用某些函数，比如在构造单例对象的时候，我们会希望该单例类的拷贝构造和赋值构造函数无法被用户调用。
在C++98时代，我们需要运用将该函数定义为私有函数但是不实现它的技巧来实现这个目的，如果访问到这些函数，在链接的时候会因为找不到定义而引起链接错误。
<!-- more -->
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios :public ios_base {
public:
	...
private:
	basic_ios(const basic_ios& );                   // 没有定义
	basic_ios& operator=(const basic_ios&);         // 没有定义
};
```
而C++11提供了delete关键字来更好的完成这个任务。
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    ...
    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    ...
};
```
任何代码试图调用该函数都会引发一个编译错误，而不是链接错误，这是delete关键字的优势之一。

delete关键字的另一个优势是，任何函数都可以被标记为delete，而私有函数首先必须是个类的成员函数。比如我们有一个判断一个int是否为幸运数字的简单函数，
```cpp
bool isLucky(int number);
```
因为C++中很多类型都能隐式转换为int，所以以下调用也是合法的，
```cpp
if(isLucky('a')) ...              // a 是否是幸运数字？
if(isLucky(ture)) ...             // 返回true?
if(isLucky(3.5)) ...              // 我们是否应该在检查它是否幸运之前裁剪为3？
```

但是有些转换不是我们期望的，这时就可以将不需要的重载标记为delete，
```cpp
bool isLucky(int number);           // 原本的函数
bool isLucky(char) = delete;        // 拒绝char类型
bool isLucky(bool) = delete;        // 拒绝bool类型
bool isLucky(double) = delete;      // 拒绝double和float类型
```

delete还可以用来禁止特定的模板实现，这也是私有函数无法做到的。
```cpp
template<typename T>
void processPointer(T* ptr);
```
在指针的家族中，有两个特殊的指针。一个是void* 指针，因为没有办法对它们解引用，递增或者递减它们。另一个是char* 指针，因为它们往往表示指向C类型的字符串，而不是指向独立字符的指针。这些特殊情况经常需要特殊处理，在processPointer模板中，我们希望能拒绝这两种指针，可以用以下方法来简单的实现，
```cpp
template<>
void processPointer<const void>(const void*) = delete;

template<>
void processPointer<const char>(const char*) = delete;
```
而以下的方法则行不通，
```cpp
class Widget{
public:
    ...
    template<typename T>
    void processPointer(T* ptr)
    { ... }

private:
    template<>                                       // 错误！
    void processPointer<void>(void*)
};
```
因为一个成员函数模板的某个偏特化，拥有不同于模板主体的访问权限是不被允许的。

### 总结
- 优先使用删除函数而不是私有而不定义的函数
- 任何函数都可以被声明为删除，包括非成员函数和模板函数
