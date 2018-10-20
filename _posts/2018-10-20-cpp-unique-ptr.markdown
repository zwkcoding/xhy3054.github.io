---
layout: post
title: c++智能指针－－unique_ptr
date: 2018-10-20 10:07:24.000000000 +09:00
img:  cute2.jpg # Add image post (optional)
tag: [c与c++]
---

众所周知，c语言与c++需要自行管理动态的内存。许多代码写到最后，防止内存泄漏需要花费程序员大量的心力。在c++11标准中，提出了智能指针来帮助程序员**管理动态内存**(如果使用智能指针来管理栈上的内存，则在最后析构阶段释放内存时会出现错误)。智能指针主要有三种，分别是`shared_ptr`、`unique_ptr`与`weak_ptr`。本文主要讲的就是`unique_ptr`使用中的问题。

# unique_ptr
与`shared_ptr`不同，某个时刻只能有一个`unique_ptr`指向其管理的动态内存上的对象。当这个`unique_ptr`销毁时，它所指向的对象也会被销毁。

### 原理分析
```cpp
namespace std {
	template <typename T, typename D = default_delete<T>>
	class unique_ptr
	{
	public:
		explicit unique_ptr(pointer p) noexcept;	
		~unique_ptr() noexcept;    
		T& operator*() const;
		T* operator->() const noexcept;
		unique_ptr(const unique_ptr &) = delete;
		unique_ptr& operator=(const unique_ptr &) = delete;
		unique_ptr(unique_ptr &&) noexcept;	//右值引用
		unique_ptr& operator=(unique_ptr &&) noexcept;
		// ...
	private:
		pointer __ptr;
	};
}
```
***
由上源代码我们可以了解到：
1. `unique_ptr`内部存储一个内置指针，当`unique_ptr`析构时，它的析构函数将会负责析构它持有的对象。

2. `unique_ptr`提供了`operator*()`和`operator->()`成员函数，像内置指针一样，我们可以使用 * **解引用**unique_ptr，使用 -> 来**访问**unique_ptr所持有对象的成员。

3. `unique_ptr`并不提供 **copy** 操作，这是为了防止多个`unique_ptr`指向同一对象。

4. 但`unique_ptr`提供了 **move** 操作，因此我们可以用`std::move()`来转移unique_ptr。

5. 两种构造函数，一种传入内置指针，一种传入`unique_ptr`的右值。第一种是`explicit`的，第二种不是（所以在函数传参时，赋值时～）。

### 构造选择
与`shared_ptr`不同，并没有类似`make_shared()`的标准库函数返回一个`unique_ptr`。当我们定义一个unique_ptr时，**只能使用直接初始化**，参数是**一个指向动态内存的内置指针或者一个`unique_ptr`类型的右值引用（通过 move 获得）**。
```cpp
unique_ptr<string> p1(new string("hello~"));
unique_ptr<string> p2(std::move(p1));	//p1管理的动态内存转移到p2，p1可以正常释放，但不再可用
```

### 销毁操作
缺省情况下，`unique_ptr`会使用`delete`析构对象，试用于new申请的动态内存。如果不适用，我们可以使用自定义的 deleter。
```cpp
struct Widget{  };
// ...
auto deleter = []( Widget *p ) {
    cout << "delete Widget!" << endl;
    delete p;
};
unique_ptr<Widget, decltype(deleter)> ptr{ new Widget, deleter };	//注意！此处模板需要指定删除器的类型，这一点与shared_ptr不同
```
***
我们发现，此处模板**需要指定删除器的类型**，这一点与shared_ptr不同。当然，我们可以使用 C++11 的 alias template 特性，这样就可以避免指定 deleter 的类型：
```cpp
struct Widget{  };
template <typename T>
using uniquePtr = unique_ptr<T, void(*)(T*)>;
void func()
{
    uniquePtr<Widget> ptr( new Widget, 
                           []( Widget *p ) {
                               cout << "delete Widget!" << endl;
                               delete p;
                           });
}
```
***

除此之外，`unique_ptr`为数组提供了**模板偏特化**，因此unique_ptr也可以指向数组，下为`unique_ptr`源码：
```cpp
namespace std {
    template <typename T, typename D>
    class unique_ptr<T[], D>
    {
    public:
        // ...
        T& operator[]( size_t i ) const;
    };
    template <typename T>
    class default_delete<T[]>
    {
    public:
        // ...
        void operator()( T *p ) const;    // call delete[] p
    };
}
```
***
当`unique_ptr`指向数组时，可以使用`[]`来访问数组元素。default_delete也为数组提供模板偏特化，因此当unique_ptr被销毁时，会调用`delete []`释放数组内存。
```cpp
unique_ptr<string[]> ptr{ new string[100] };
ptr[0] = "hello";
ptr[1] = "world";
```
### 使用unique_ptr的坑－－需要注意的问题1
`unique_ptr`是用来独占地持有对象的，所以通过同一原生指针来初始化多个`unique_ptr`，下面是一种错误的使用方式：
```cpp
struct Widget{  };
Widget *ptr = new Widget;
unique_ptr<Widget> p1{ ptr };
unique_ptr<Widget> p2{ ptr };     // ERROR: multiple ownership
```
***
当p1和p2各自被销毁的时候，它们指向的Widget将被delete两次。

### 使用unique_ptr的坑－－需要注意的问题2
不能拷贝或赋值一个`unique_ptr`，但是可以拷贝或赋值一个**将要被销毁**的`unique_ptr`。最常见的例子是从函数返回一个`unique_ptr`，还有在函数实参中构造`unique_ptr`。在这种情况下，编译器知道这个对象将要销毁，将会执行一种特殊的“拷贝”（其实是**移动操作**）。

### 使用unique_ptr的坑－－需要注意的问题2
使用`unique_ptr`**并不能绝对地保证异常安全**，你可能很惊讶于这个结论。让我们看看一个例子：
```cpp
func(unique_ptr<T>{ new T }, func_throw_exception());
```
***
C++ 标准并没有规定编译器对函数参数的求值次序，所以有可能出现这样的次序：
1. 调用`new T`分配动态内存。
2. 调用`func_throw_exception()`函数。
3. 调用`unique_ptr`的构造函数。

调用`func_throw_exception()`函数会抛出异常，所以无法构造`unique_ptr`，导致`new T`所分配的内存不能回收，造成了内存泄露。解决这个问题，最好有一个`make_unique`函数（原子操作）。但是c++11标准库中并没有提供。好消息是c++14中提供了。





