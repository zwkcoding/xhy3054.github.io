---
layout: post
title: c++中关于命名空间中成员的使用
date: 2018-11-07 10:07:24.000000000 +09:00
img:  cute8.jpg # Add image post (optional)
tag: [c与c++]
---

使用c++进行开发的我们总会不可避免的使用很多外部的库，这样可以避免自己重复造轮子，大大提高工作效率。但是每个库都会定义大量的全局名字，如类、函数、模板等。如果这些名字都放在全局命名空间中，不可避免的会出现相同的名字，形成**命名冲突。**

**命名空间**就是为了防止命名冲突而出现的，每个库都可以拥有自己的命名空间，每个命名空间是一个作用域。然后如何使用一个特定命名空间里的成员，就是本文主要要讲的内容，不过我只挑了一些我认为需要记录下来的，因为有很多常识性的规则实在没有多说的必要。

# using声明
一条`using`声明语句一次只引入命名空间的一个成员。例如最常见的`using std::cout`就是使用using声明将std命名空间中的标准输出对象`cout`引入**当前作用域**。其实就是在当前作用域声明了有`cout`这个对象，并且说明它定义在命名空间`std`里。因此如果当前作用域中已经有一个同名的对象，将会**报错**。

需要注意的是using声明**引入的是一个名字**，所以在在引入一个**函数**时比较特殊。如下：
```cpp
using ns::print(int);	//错误：只能引入一个名字，不能指定形参列表
using ns::print;	//正确
```
---
并且在引入一个函数时，会出现如下情况。
1. 如果using声明出现在局部作用域中，则引入的名字将隐藏外层作用域的相关声明；
2. 如果using声明所在的作用域中已经有一个函数与新引入的函数同名且形参列表相同，则将引发错误；
3. 除了上面两种情况，其他同名函数都会进行**重载**，最终扩展候选函数集的规模。

# using指示
`using`指示是将某个命名空间中的所有内容都变得有效。但是有一点需要注意，那就是using指示是对**最近的外层作用域**有效。如下
```cpp
// namespace A and function f are defined at global scope
//定义在全局作用域
namespace A {
    int i = 0, j = 42;
}

void f() 
{
    using namespace A; // 把A中的名字注入到全局作用域

	// uses i and j from namespace A
    cout << "i: " << i << " j: " << j << endl;
}
```
---
并且using指示后在同一作用域出现相同名字的时候，并**不会马上报错**，而是直到使用时出现二义性的代码时才会报错。
```cpp
namespace blip {
    int i = 16, j = 15, k = 23;
	void f() 
		{ cout << "i: " << i << " j: " << j << " k: " << k << endl; }
}

int j = 0;  // ok: j inside blip is hidden inside a namespace

int main() 
{
    // using directive; 
	// the names in blip are ``added'' to the global scope
    using namespace blip; // clash between ::j and blip::j
                          // detected only if j is used

    ++i;        // sets blip::i to 17
    ++j;        //　二义性错误，是全局的j还是blip::j?
    ++::j;      // ok: sets global j to 1
    ++blip::j;  // ok: sets blip::j to 16

    int k = 97; // local k hides blip::k
    ++k;        // sets local k to 98

	return 0;
}
```
如上，如果使用using指示引入函数与最近的外层作用域重名，会直接被添加到重载集合，即使出现同名且参数列表相同的函数也不会报错，而是知道出现二义性代码才会报错。

> 通常情况下大家都是在最外层的全局作用域使用using指示，此时没有更外层的作用域了，所以会直接将成员注入到全局作用域。

# 实参相关与类类型形参的函数查找
如果不用using指示与using声明，通常我们在使用一个命名空间中的成员时，需要使用作用域限定符`::`指明出处，但是有一点例外，那就是在使用函数时。

1. 当我们调用一个函数，并且此函数的实参传递一个类类型的对象是，**除了在常规作用域进行函数查找外，还会查找实参类所属的命名空间**（比如`std::cout<<"helio"`中对于重载的输出运算符函数并没有指定作用域，但是依然找到了，便是根据实参cout）。

2. 如果形参是一个基类，传入的实参是该基类的一个派生类，则除了在常规作用域与实参类所属的命名空间中进行函数的查找外，还会在**基类的作用域中**进行该函数的查找。如下：

```cpp
namespace NS {
    class Quote { 
	public:
		Quote() { std::cout << "Quote::Quote" << std::endl; } 
	};
    void display(const Quote&) 
		{ std::cout << "display(const Quote&)" << std::endl; }
}

// Bulk_item's base class is declared in namespace NS
class Bulk_item : public NS::Quote { 
public:
	Bulk_item() { std::cout << "Bulk_item::Bulk_item" << std::endl; }
};

int main() {
    Bulk_item book1;

    display(book1); // calls Quote::display

    return 0;
}
```

# 参考资料
- [1] C++ Primer（第5版）






