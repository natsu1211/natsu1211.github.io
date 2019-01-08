---
title: 读薄Effective Modern C++ (条款13 优先使用const_iterator而不是iterator)
date: 2018-11-21 00:21:26
tags:
- c++
categories:
- [编程语言]
- [读书笔记]
---
## 条款13 优先使用const_iterator而不是iterator

const_iterator在STL中等价于指向const的指针。被指向的数值是不能被修改的。标准的做法是应该使用const的迭代器的地方，也就是尽可能的在没有必要修改指针所指向的内容的地方使用const_iterator。      
<!-- more -->
   
比如我们想在vector中找到1983并把它替换成1998，我们会写下以下代码,
```cpp
std::vector<int> values;
...
std::vector<int>::iterator it =
    std::find(values.begin(),values.end(), 1983);
values.insert(it, 1998);
```
这里因为并不会改变itertor所指向的数据，const_iterator应当是更合适的选择，然而在C++98中，std::find等函数只接受非const的iterator，并且没有简单的办法从非const容器获得const_iterator，在C++98中使用const_iterator只是自寻烦恼。     
然而情况在C++11中大有不同，const_iterator容易获取并且容易使用。
```cpp
std::vector<int> values; // as before ...
auto it = std::find(values.cbegin(),values.cend(), 1983); // and cend
values.insert(it, 1998);// use cbegin
```
需要注意的是，当你需要写出更为泛型的代码的时候，C++11对const_iterator的支持可能会略显不足，
比如我们要写一个在容器中寻找元素并进行替换的泛型函数，原生C数组，我们会希望使用非成员函数的版本来获取迭代器。
```cpp
template<typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal)
{
    using std::cbegin;
    using std::cend;
    const V& targetVal,
    const V& insertVal)
    auto it = std::find(cbegin(container), cend(container), targetVal);
    container.insert(it, insertVal);
}
```
上面的代码需要C++14才会工作，因为C++11中没有提供cbegin和cend的非成员函数版本。
我们可以自己为C++11添加，
```cpp
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    return std::begin(container);        
}
```

## 总结
- 优先使用const_iterator而不是iterator
- 在泛型代码中，优先使用非成员函数版本的begin, end, rbegin ,rend函数






