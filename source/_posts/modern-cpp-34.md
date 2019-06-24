---
title: 读薄Effective Modern C++（条款34 比起std::bind更倾向于使用lambda)
date: 2019-06-15 23:24:39
tags:
- [C++]
- [Lambda]
categories:
- [PL, Tips]
- [Memo]
---

## 条款34 比起std::bind更倾向于使用lambda

在C++11，比起使用std::bind，lambda几乎总是更好的选择。lambda通常具有更好的可读性，更强的表达能力，更好的性能。

假设我们有个函数用来设置警报,
```cpp
// 声明一个时间点的类型别名
using Time = std::chrono::steady_clock::time_point;

enum class Sound {Beep, Siren, Whistle };

// 声明一个时间长度的类型别名
using Duration = std::chrono::steady_cloak::duration;

// 在时间点t，发出警报声s，持续时间为d
void setAlarm(Time t, Sound s, Duration d);

// setSoundL是一个允许指定警报声音类型的函数对象
// 警报在一个小时后触发，持续30秒
auto setSoundL = 
    [](Sound s)
    {
        using namespace std::chrono;

        setAlarm(steady_clock::now() + hours(1),
                 s,
                 seconds(30));
    };
```
<!--more-->

实现同样功能的bind对象需要这样写，为了将`steady_clock::now() + 1h`的求值推迟到setAlarm调用时，需要借助std::plus，这增加了语法噪音，降低了可读性。
```cpp
using namespace std::chrono;
using namespace std::placeholders;

auto setSoundB = 
    set::bind(setAlarm,
              std::bind(std::plus<steady_clock::time_point>(),
                        steady_clock::now(),
                        hours(1)),
              _1,
              seconds(30));
```

再来考虑表达能力，如果我们考虑为setAlarm函数增加一个重载，接受第四个参数来指定警报的音量：
```cpp
enum class Volume { Normal, Loud, LoudPlusPlus };
void setAlarm(Time t, Sound s, Duration d, Volume v);
```

lambda版本依然工作良好，因为它的setAlarm调用会重载到三个参数的版本，而std::bind版本则会出现编译错误，问题在于编译器没有办法决定哪个setAlarm应该被传递给std::bind，它拥有的只是一个函数名，而这单独的函数名是有歧义的。

为了让std::bind版本通过编译，setAlarm必须转换为合适的函数指针类型，

```cpp
using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);

auto setSoundB =
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm),
              std::bind(std::plus<>(),
                        steady_clocl::now()
                        1h),
              _1,
              30s);
```

不过，这意味着在setSoundB函数内，将以函数指针的方式调用setAlarm，而那意味着通setSoundB调用的setAlarm，比通过setSoundL调用的setAlarm进行内联的可能性更低。因此，使用lambda生成的代码可能会比使用std::bind的快。

结论就是，在C++14，没有理由使用std::bind。


## 总结

- 比起使用std::bind，lambda有更好的可读性，更强的表达能力，可能还有更高的效率。

