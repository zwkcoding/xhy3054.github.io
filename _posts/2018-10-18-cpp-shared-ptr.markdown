---
layout: post
title: c++智能指针－－shared_ptr
date: 2018-10-18 10:07:24.000000000 +09:00
img:  cute1.jpg # Add image post (optional)
tag: [c与c++]
---

众所周知，c语言与c++需要自行管理动态的内存。许多代码写到最后，防止内存泄漏需要花费程序员大量的心力。在c++11标准中，提出了智能指针来帮助程序员**管理动态内存**。智能指针主要有三种，分别是`shared_ptr`、`unique_ptr`与`weak_ptr`。本文主要讲的就是`shared_ptr`使用中的问题。

# shared_ptr
首先需要说明的是，`shared_ptr`是一个**模板类**。为了让用户可以像使用内置指针一样使用它，这个类的设计者为**它重载了解应用运算符`*`、成员访问运算符`->`、赋值运算符`=`还有向bool类型的显式类型转换**等。

我们都知道，在使用`new`动态申请一块内存ａ后会返回一个指向这片内存的指针，对这个指针值进行拷贝赋值等操作，最后会出现很多个指针都指向ａ。但是通过指针释放ａ只能释放一次，如果释放了多次，就会出现错误。以前，大都是通过自己创建一种**引用计数机制**来管理这样的情况。

在c++11中，标准库提供了一种`shared_ptr`智能指针类型。一个动态分配的对象可以在多个`shared_ptr`之间共享，因此，`shared_ptr`支持拷贝操作。
```cpp
shared_ptr<string> ptr1{ new string("hello") };
auto ptr2 = ptr1;	//拷贝构造，ptr1与ptr2指向同一块动态内存（一个string对象）
```

### 原理分析
`shared_ptr`内部包含两个指针，一个指向对象，另一个指向控制块(control block)，控制块中包含一个引用计数和其它一些数据。由于这个控制块需要在多个shared_ptr之间共享，所以它也是存在于`heap`中的。`shared_ptr`对象本身是线程安全的，也就是说`shared_ptr`的引用计数增加和减少的操作都是原子的。

![sh2]({{site.baseurl}}/assets/img/shared_ptr/sh2.png)

### 构造选择
在初始化`ptr`时一般有两种选择，分别是`new`与`make_shared`。
```cpp
shared_ptr<std::string> ptr{ new string("hello") };
auto ptr = std::make_shared<std::string>("hello");
```
***
推荐使用第二种。因为第一种方式创建`shared_ptr`时，需要执行两次new操作，一次在 heap 上为 string("hello") 分配内存，另一次在 heap 上为控制块分配内存。使用`make_shared`来创建`shared_ptr`会高效，因为`make_shared`仅使用new操作一次，它的做法是在 heap 上分配一块连续的内存用来容纳 string("hello") 和控制块。同样，当`shared_ptr`被析构时，也只需一次`delete`操作。

### 销毁操作
一般情况下不需要考虑所指对象的销毁问题，只在指向数组时，与`unique_ptr`不同，标准库并不提供`shared_ptr<T[]>`，因此，使用`shared_ptr`处理数组时需要显示指定删除行为，例如：
```cpp
shared_ptr<string> ptr1( new string[10], 
                         []( string *p ) {
                             delete[] p;
                         });
shared_ptr<string> ptr2( new string[10],
                         std::default_delete<string[]>() );
```
***
> 此处需要说明的还有，一般智能指针管理的资源都是new分配的内存（即默认使用delete进行销毁操作），如果不是，比如说是其他方式申请的动态内存或者申请的不是动态内存等，请记住传递给它一个删除器来订制销毁操作。

### 使用shared_ptr的坑－－需要注意的问题１
用同一个内置指针初始化`shared_ptr`的操作只能出现一次。
```cpp
int *p = new int{10};
shared_ptr<int> ptr1{ p };
shared_ptr<int> ptr2{ p };         // ERROR
```
很明显，每次通过内置指针来构造`shared_ptr`时候就会分配一个控制块，这时存在两个控制块，也就是说存在两个引用计数。这显然是错误的，因为当这两个`shared_ptr`被销毁时，对象将会被delete两次。

### 使用shared_ptr的坑－－需要注意的问题２
在对象之间出现**循环引用**时，会使得共享指针引用计数不会降到０，也就不能销毁。循环引用示意图如下：

![am]({{site.baseurl}}/assets/img/shared_ptr/am.png)

```cpp
#include <iostream>
#include <memory>
using std::shared_ptr;
using std::make_shared;
// 一段内存泄露的代码
struct Son;
struct Father{
    shared_ptr<Son> son_;
};
struct Son{
    shared_ptr<Father> father_;
};
int main() 
{
    auto father = make_shared<Father>();
    auto son = make_shared<Son>();
    father->son_ = son;
    son->father_ = father;
    std::cout<<"one father's son:"<<father.use_count()<<std::endl;  
    std::cout<<"one son's father:"<<son.use_count()<<std::endl;  
    return 0;
}
```
***
编译运行结果为：
```bash
xhy@ubuntu:~/cpp_learn/share_ptr$ ./test 
one father's son:2
one son's father:2
```
函数结束前，堆上的两个对象的引用计数都是２，所以即便函数结束，将两个栈上的的共享指针分别析构，最后堆上的两个对象的引用数也不会为０，而是１，两个对象不会调用析构函数进行析构，从而内存泄漏。

# 参考资料
- [1] C++ Primer（第5版）
- [2] http://senlinzhan.github.io/2015/04/24/%E6%B7%B1%E5%85%A5shared-ptr/


