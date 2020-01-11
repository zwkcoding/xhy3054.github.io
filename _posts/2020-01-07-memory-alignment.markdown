---
layout: post
title: SSE/AVX加速时的内存对齐问题
date: 2020-01-07 10:07:24.000000000 +09:00
img:  one-piece/one-piece22.jpg # Add image post (optional)
tag: [tools]
---
上个月比较忙，一篇博客都没写，这是2020第一篇～

> 这篇博客的内容主要参考了官网与[一篇知乎上的文章](https://zhuanlan.zhihu.com/p/93824687)

# 内存对齐问题

这个是在学习Eigen的过程中顺便学习的。Eigen中有较多的矩阵与向量运算，因此可以使用SSE、AVX等指令集进行加速，当编译时打开`-march=native`这个选项时，会尝试对Eigen中的运算进行加速。而加速时内存对齐，因此如果不满足就会报错。

## 向量化运算
向量化运算就是用SSE、AVX等SIMD（Single Instruction Multiple Data）指令集，实现一条指令对多个操作数的运算，从而提高代码的吞吐量，实现加速效果。SSE是一个系列，包括从最初的SSE到最新的SSE4.2，支持同时操作16 bytes的数据，即4个float或者2个double。AVX也是一个系列，它是SSE的升级版，支持同时操作32 bytes的数据，即8个float或者4个double。

但向量化运算是有前提的，那就是内存对齐。SSE的操作数，必须16 bytes对齐，而AVX的操作数，必须32 bytes对齐。也就是说，如果我们有4个float数，必须把它们放在连续的且首地址为16的倍数的内存空间中，才能调用SSE的指令进行运算。

> 关于SSE与AVX还有SIMD的介绍附在了文后。至于内存对齐的原因，尽管内存是以字节为单位，但是大部分处理器并不是按字节块来存取内存的.它一般会以双字节,四字节,8字节,16字节甚至32字节为单位来存取内存，我们将上述这些存取单位称为内存存取粒度.显然此处向量化运算的存取粒度会比较大一点。

### 简单的AVX加速示例

```cpp
#include <immintrin.h>
#include <iostream>

int main() {

  double input1[4] = {1, 1, 1, 1};
  double input2[4] = {1, 2, 3, 4};
  double result[4];

  std::cout << "address of input1: " << input1 << std::endl;
  std::cout << "address of input2: " << input2 << std::endl;

  __m256d a = _mm256_load_pd(input1);
  __m256d b = _mm256_load_pd(input2);
  __m256d c = _mm256_add_pd(a, b);

  _mm256_store_pd(result, c);

  std::cout << result[0] << " " << result[1] << " " << result[2] << " " << result[3] << std::endl;

  return 0;
}
```

> 这段代码使用AVX中的向量化加法指令，同时计算4对double的和。这4对数保存在input1和input2中。 `_mm256_load_pd`指令用来加载操作数，`_mm256_add_pd`指令进行向量化运算，最后， `_mm256_store_pd`指令读取运算结果到`result`中。可惜的是，程序运行到第一个`_mm256_load_pd`处就崩溃了。崩溃的原因正是因为输入变量没有内存对齐。我特意打印出了两个输入变量的地址，结果如下 

```bash
address of input1: 0x7ffeef431ef0
address of input2: 0x7ffeef431f10 
```

上一节提到了AVX要求32字节对齐，我们可以把这两个输入变量的地址除以32，看是否能够整除。结果发现`0x7ffeef431ef0`和`0x7ffeef431f10`都不能整除。当然，其实直接看倒数第二位是否是偶数即可，是偶数就可以被32整除，是奇数则不能被32整除。

如何让输入变量内存对齐呢？我们知道，对于局部变量来说，它们的内存地址是在编译期确定的，也就是由编译器决定。所以我们只需要告诉编译器，给`input1`和`input2`申请空间时请让首地址32字节对齐，这需要通过预编译指令来实现。不同编译器的预编译指令是不一样的，比如gcc的语法为`__attribute__((aligned(32)))`，MSVC的语法为 `__declspec(align(32))` 。以gcc语法为例，做少量修改，就可以得到正确的代码

```cpp
#include <immintrin.h>
#include <iostream>

int main() {

  __attribute__ ((aligned (32))) double input1[4] = {1, 1, 1, 1};
  __attribute__ ((aligned (32))) double input2[4] = {1, 2, 3, 4};
  __attribute__ ((aligned (32))) double result[4];

  std::cout << "address of input1: " << input1 << std::endl;
  std::cout << "address of input2: " << input2 << std::endl;

  __m256d a = _mm256_load_pd(input1);
  __m256d b = _mm256_load_pd(input2);
  __m256d c = _mm256_add_pd(a, b);

  _mm256_store_pd(result, c);

  std::cout << result[0] << " " << result[1] << " " << result[2] << " " << result[3] << std::endl;

  return 0;
}
```

输出结果为
```bash
address of input1: 0x7ffc5ca2e640
address of input2: 0x7ffc5ca2e660
2 3 4 5
```

可以看到，这次的两个地址都是32的倍数，而且最终的运算结果也完全正确。

### 动态内存申请时的对齐问题

> 上面的操作是在栈上进行地址对齐的内存申请，此时内存地址是编译器在编译时确定的，因此预编译指令可以生效。但是我们在真正的编程时，经常会使用到动态内存申请，动态创建的对象存储在堆上，其地址是运行时确定的。

其实要想实现动态申请内存的32位地址对齐，方法有很多
1. 为该类重写`new`函数，每次分配内存时多申请32个字节，然后寻找这段内存中第一个32倍数的地址返回。

2. c++从C++11开始新增了函数`void *aligned_alloc( size_t alignment, size_t size );`可以动态申请到alignment字节对齐的内存;

3. 从C++17开始新增了函数`void* operator new ( std::size_t count, std::align_val_t al);`函数可以动态申请对齐的内存;

> 使用`new`动态创建一个类对象，首先使用`new`在堆上申请该类大小的内存，然后调用该类的构造函数在申请的内存上构造该对象。所以如果想使用`new`动态申请一个内存对齐的对象，关键在于第一步，方法是1。（当然了在重写该类的new函数可以直接使用2和3来实现，也可以自行判断保证返回的指针是内存对齐的）

## Eigen中的内存对齐方式

在Eigen官方文档中说，如果你想要创建一个类，然后这个类中包含Eigen类型，那么需要考虑内存对齐的问题。比如如下：

```cpp
class Foo
{
  ...
  Eigen::Vector2d v;
  ...
};
...
Foo *foo = new Foo;
```

> 上面这个类中含有一个[固定尺寸的Eigen对象](http://eigen.tuxfamily.org/dox/group__TopicFixedSizeVectorizable.html)，所以需要考虑内存对齐。解决方法如下：

```cpp
class Foo
{
  ...
  Eigen::Vector2d v;
  ...
public:
  EIGEN_MAKE_ALIGNED_OPERATOR_NEW
};
...
Foo *foo = new Foo;
```

> 只需要在你的`class`的public部分添加一个宏定义声明`EIGEN_MAKE_ALIGNED_OPERATOR_NEW`即可。这个宏定义其实就是一个`void* operator new()`的重载函数，这个重载的函数保证返回一个内存对齐的指针，Eigen将其封装成了一个宏，因此用户只要在此处添加这个宏，即代表着重写了这个类的new函数。


## SIMD(单指令多数据)
SIMD单指令流多数据流(SingleInstruction Multiple Data,SIMD)是一种采用一个控制器来控制多个处理器，同时对一组数据（又称“数据向量”）中的每一个分别执行相同的操作从而实现空间上的并行性的技术。在微处理器中，单指令流多数据流技术则是一个控制器控制多个平行的处理微元，例如Intel的MMX或SSE以及AMD的3D Now!技术还有RAM的NEON等。

> 所以SIMD其实是一种技术，目前的处理器上基本都会支持SIMD，只是需要相关的指令集来使用它。SSE与AVX都是intel提出的适用于自己处理器的SIMD指令集。AVX是SSE的延伸架构。其他公司比如AMD或者ARM也都有推出自己的SIMD指令集。想了解的更详细可以自己再查阅相关资料。


# 参考文献
- [1] Eigen官网
- [2] https://zhuanlan.zhihu.com/p/93824687
