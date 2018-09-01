---
layout: post
title: "c语言中的变长数组与零长数组"
date: 2018-08-29 10:07:24.000000000 +09:00
img:  vally.jpg # Add image post (optional)
tag: [c与c++]
---

编程确实是需要在实践中提高，这段时间一直在学习ｃ++，同时刷刷leetcode。遇到了很多问题，也搞清楚了很多原本模糊不清的概念，今天主要写一下ｃ语言中的变长数组与零长数组。尤其是其中的变长数组，可是困扰了我一段时间了。

# c语言中的变长数组
想必很多学习c++的人都会在书上看到，数组在初始化时一定得确定维度，也就是说定义数组时，维度一定要用常量。但是在编程中很多人肯定发现了，如下程序也能正常运行：
```cpp
#include<iostream>
using namespace;
int main(){
	int size;
	cin>>size;
	int data[size];
	for (int i=0; i<size; i++)
		cin>>data[i];
	return 0;
}
```
这是怎么回事？难道以前我学的是错的吗？当然不是。这个问题需要仔细说明一下。

在ｃ++的标准中确实规定了：定义数组时，元素个数必须确定，因此维度一定要用常量。上面的程序之所以能编译运行成功，是因为**c++11对c99的编译器扩展**。

> c99中引入了变长数组的概念，在[ｃ99的技术手册](https://gcc.gnu.org/onlinedocs/gcc/Variable-Length.html)中说明了数组的长度可以为变量的，称为变长数组（VLA，variable length array）。（注：这里的变长指的是数组的长度是在运行时才能决定，但一旦决定在数组的生命周期内就不会再变。）

> 在 GCC 标准规范的 6.19 Arrays of Variable Length 中指出，作为编译器扩展，GCC 在 C90 模式和 C++ 编译器下遵守 ISO C99 关于变长数组的规范。

变长数组肯定是有好处的，它可以实现与`alloca`一样的效果，在栈上进行动态的空间分配，并且在函数返回时自动释放内存，无需手动释放。但是记住，**这个是c99的标准，不是c++的**，也就是说有的ｃ++编译器上并不支持。而且**谷歌的c++编程规范**中说:

![c++]({{site.baseurl}}/assets/img/array_length/googlecpp.png)


> 注意：其实变长数组与`alloca()`函数，这两种在栈空间上进行动态的内存分配本身是很酷的一件事，这部分内存函数返回时自动释放，而且运行效率很高。但是目前并未得到广泛引用，而且在很多平台上没有得到支持，因此可移植性很差。因此，不推荐在一些项目开发中使用，平时刷刷leetcode时使用倒是无伤大雅。

# c语言中的零长数组
> 首先说明，零长数组在ISO C与C++的规格说明书中是不允许的，gcc 为了预先支持C99的这种玩法，所以，让“零长度数组”这种玩法合法了。关于GCC对于这个事的文档在这里：“[Arrays of Length Zero](https://gcc.gnu.org/onlinedocs/gcc/Zero-Length.html)”。

```c
struct line{
	int length;
	char data[0];
}
```
如上定义了一个结构体，其中第二个元素为一个长度为零的数组。

学习过ｃ语言的同学都知道数组，但是长度为０的数组你们可能很少看到。这样有什么用处呢？其实这样做是为了使得这个结构体的长度可以随着程序动态的变化，十分的灵活，比如如下形式使用这个结构体：
```c
struct line *exa = (struct line *)malloc(sizeof(struct line)+10);
exa->length = 10;
```
其中，第一个元素`length`	标志着下面数据的长度。`sizeof(struct line)`的结果为`sizeof(int)`，因为另一个元素零长数组不占空间，**数组名data是一个指针常量，标识着成员变量`length`后面的那个地址，但其本身并不占空间**。

这与下面这种形式的结构体是有区别的。
```c
struct line{
	int length;
	char *data;
}
```
这个结构体中，`sizeof(struct line)`的结果为`sizeof(int)+sizeof(char *)`。

第一种的方法会有一些好处：
1. 方便内存释放。比如在我们为别人提供的函数接口中返回了一个结构体。如果是第二种结构体，其实在结构体的构造中是进行了两次的内存分配的，一次是为line这个结构体进行的内存申请，一次是为`line->data`这个指针指向的内存进行的内存申请。这时我们如果要进行这部分的内存释放，其实是需要分先后进行两次内存释放的。但是如果是第一种结构体，只需要释放(free)一次就行。本来对于这个结构体的构建者来说，这不是什么问题，但是对于仅仅是调用这个函数获取一个结构体的人来说，他们会方便很多。而且我们也不能指望着用户自己发现需要对这个数据结构进行两次free。
2. 这样有利于访问速度。连续的内存有益于提高访问速度，也有益于减少内存碎片。（其实，我个人觉得也没多高了，反正你跑不了要用做偏移量的加法来寻址）

最后附一个完整代码：
```c
#include <stdlib.h>
#include <string.h>
 
struct line {
   int length;
   char contents[0]; // C99的玩法是：char contents[]; 没有指定数组长度
};
 
int main(){
    int this_length=10;
    struct line *thisline = (struct line *)malloc (sizeof (struct line) + this_length);
    thisline->length = this_length;
    memset(thisline->contents, 'a', this_length);
    return 0;
}
```
上面这段代码的意思是：我想分配一个不定长的数组，于是我有一个结构体，其中有两个成员，一个是length，代表数组的长度，一个是contents，代表数组的内容。后面代码里的this_length（长度是10）代表是我想分配的数据的长度。（这看上去是不是像一个C++的类？）这种玩法英文叫：Flexible Array，中文翻译叫：**柔性数组**。






