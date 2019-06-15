---
title: 读薄Effective Modern C++（条款31 避免使用lambda表达式的默认捕获模式）
date: 2019-06-09 02:46:13
tags:
- [C++]
- [Lambda]
categories:
- [PL, Tips]
- [Memo]
---

## 条款31 避免使用lambda表达式的默认捕获模式

C++11中有两种默认捕获模式：引用捕获或值捕获。
引用捕获可能导致空悬引用。而值引用一样有捕获空悬指针的可能，并且更为隐蔽。应当避免使用默认捕获，而使用显式捕获。
引用模式捕获的危害显而易见，你可能会捕获一个局部变量，但当局部变量声明周期结束时，lambda捕获的局部变量引用就会变成空悬引用。

```cpp
int divisor = 5;

using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;               // 含有过滤函数的容器 
filters.emplace_back(
    [&](int value) { return value % divisor == 0; }   // 危险！
);
```
虽然使用显式捕获无法解决这个问题，但是会提醒我们divisor变量的生命周期应当至少不短于lambda表达式的生命周期。

<!--more-->

使用值捕获能够解决这个问题，
```cpp
filters.emplace_back(            // 现在divisor不会空悬
  [=](int value)  { return value % divisor == 0; }
```

但是也并不是总能防止空悬发生的，比如
```cpp
class Widget {
public:
    ...                      // 构造函数等
    void addFilter() const;  // 添加一个条目

private:
    int divisor;
};

void Widget::addFilter() const
{
    filters.emplace_back(
      [=](int value) { return value % divisor == 0; }
    );
}
```
捕获只能用于可见的非static局部变量（包含形参）。在Widget::addFilter内部，divisor不是局部变量，而是Widget类的成员变量，它是不能被捕获的，如果默认捕获模式被删除，代码就不能编译了。你也许会奇怪为什么默认捕获的版本能够通过编译，原因在于它捕获的不是divisor而是this指针，因此函数相当于
```cpp
void Widget::addFilter() const
{
    filters.emplace_back(
      [=](int value) { return value % this->divisor == 0; }
    );
}
```
也就是说lambda表达式是否能够如期工作取决于Widget对象的，this指针就变成了一个空悬指针。使用默认值捕获容易引起误解，并且产生让人难以发现指针空悬的问题。

通过将希望捕获的成员变量拷贝到局部变量中，然后捕获这个局部拷贝，就可以解决这个问题了，
```cpp
void Widget::addFilter() const 
{
    auto divisorCopy = divisor;
    
    filters.emplace_back(
      [divisorCopy](int value)
      { return value % divisorCopy == 0; }
    );
}
```
在C++14中，捕获成员变量一种更好的方法是使用generalized lambda capture(见条款32),
```cpp
void Widget::addFilter() const 
{
    filters.emplace_back(               // C++14
      [divisor = divisor](int value)    // 在闭包中拷贝divisor
      { return value % divisor == 0; }  // 使用拷贝
    );
}
```

## 总结
- 默认引用捕获会导致空悬引用。
- 默认值捕获依然会受到空悬指针（尤其是this指针）的影响，而且它会使用户误认为lambda是独立的。
