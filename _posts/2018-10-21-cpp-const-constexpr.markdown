---
layout: post
title: c++中const与constexpr关键字
date: 2018-10-20 10:07:24.000000000 +09:00
img:  cute3.jpg # Add image post (optional)
tag: [c与c++]
---

本文主要讨论const变量与constexpr变量的初始化时刻与constexpr函数的编译期计算问题。[转载](http://senlinzhan.github.io/2015/05/01/%E8%B0%88%E8%B0%88C-%E7%9A%84%E7%BC%96%E8%AF%91%E5%99%A8%E8%AE%A1%E7%AE%97/)

# const变量的初始化时刻
`const`修饰变量，表示这个变量是不可修改的，因此const变量必须初始化，一经初始化就不可修改。
1. 如果const变量的初始化值是在编译时就可以确定，则在编译时初始化；

2. 如果const变量的初始化值是在运行时才确定，则在运行时初始化；

```cpp
const int SIZE = 100;
```
***
由于SIZE的值是在编译时就已经确定的，编译器会使用常量 100 来替代程序中出现的SIZE。

```cpp
vector<int> v;
const int i = v.size();
```
***
运行时初始化。

# constexpr
## constexpr函数的编译期计算
> constexpr函数则与编译期计算有关，要是constexpr函数所使用的变量其值能够在编译时就确定，那么constexpr函数就能在编译时执行计算。另一方面，要是constexpr函数所使用的变量其值只能在运行时确定，那么constexpr就和一般的函数没区别。

C++11 要求constexpr函数不能多于一条语句，但是碰到 if-else 语句时，但可以巧妙地使用条件操作符来替代：
```cpp
constexpr unsigned long long fib( unsigned n ) noexcept
{
    return ( n == 0 || n == 1 ? n : fib( n - 1 ) + fib( n - 2 ) );
}
```
***
C++14 中则放松了这个要求：
```cpp
constexpr unsigned long long fib( unsigned n ) noexcept
{
    if( n == 0 || n == 1 )
    {
        return n;
    }
    else
    {
        return fib( n - 1 ) + fib( n - 2 );   
    }
}
```
***
要是我们传递一个编译时常量给fib()，那么fib()在程序编译的时候就已经执行好了。代价是增加编译时间，但程序能执行得更高效。



## constexpr变量
const变量的值可以在编译时或运行时确定，与const相比，constexpr的限制更多，因为**constexpr变量的值必须在编译时就能确定**。

在一些场合之下，变量的值要求是编译期就必须确定的，constexpr变量正好满足要求。比如数组的大小、`std::array`的大小、`std::bitset`的大小等。
```cpp
constexpr auto SIZE = 100;
std::array<int, SIZE> arr;
```
***
定义constexpr变量的时候，变量的类型**只能是基本数据类型**、指针和引用，而不能是其它标准库类型。
```cpp
// error: constexpr variable cannot have non-literal type
constexpr string str = "hello";
```
***
不过，对于我们自己定义的类型没有这个限制，因为 constructor 和成员函数可以是constexpr函数：
```cpp
class Point
{
public:
    constexpr Point( double x = 0, double y = 0 )
        : x_( x ), y_( y )
    {  }
    
    constexpr double x() const noexcept {  return x_;  }
    constexpr double y() const noexcept {  return y_;  }
    void set_x( double x ) noexcept {  x_ = x;  }
    void set_y( double y ) noexcept {  y_ = y;  }
private:
    double x_, y_;
};
```
***
要是我们使用编译期常量来初始化Point对象，那么，在编译的时候编译器就已经创建了这个对象：
```cpp
constexpr Point pt( 10, 20 );     // Evaluate at compiling time
```
***
在 C++11 中，constexpr函数隐式地是const函数，所以你会发现set_x()和set_y()这两个函数不能是constexpr函数。

但在 C++14 中，这个限制放宽了，也就是说这两个函数可以声明为constexpr函数：
```cpp
class Point
{
public:
    // ...
    constexpr void set_x( double x ) noexcept {  x_ = x;  }
    constexpr void set_y( double y ) noexcept {  y_ = y;  }
    // ...
};
```
***





