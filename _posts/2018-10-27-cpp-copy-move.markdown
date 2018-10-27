---
layout: post
title: c++中的移动操作
date: 2018-10-27 10:07:24.000000000 +09:00
img:  cute4.jpg # Add image post (optional)
tag: [c与c++]
---

在此需要强调一点，c++中的移动操作不是指标准库函数`move`，它仅仅只是一个辅助移动操作顺利进行的标准库函数而已，功能十分单一。

# 对象的移动
本文我们讲的移动是指对象的移动操作，与拷贝操作相对等的一种操作。

> 一般而言，移动操作的目的是将目标源对象管理的资源直接**移动**到本对象，即本对象接管了原本目标对象管理的资源（而拷贝操作一般只是将资源内容复制了一份），因此移动操作往往比拷贝操作效率更高。不过这也就意味着**源目标对象的资源不再可用**。所以一般情况下这种操作都要确保源目标对象在经历了资源被窃取的情况下依旧可以正常的**生老病死**，即依然可以正常进行赋值、析构等操作。

也是因为上述拷贝与移动操作的差别，有一些特殊的对象（IO类对象、unique_ptr类对象等）**只支持移动操作而不支持拷贝操作**，比如标准输入输出对象cout与cin，这是因为输入/输出缓冲区只有一个，不可能有多个对象都管理它。

## 移动构造函数与移动赋值运算符
相比于移动构造函数与移动赋值运算符，我们更熟悉的可能是拷贝构造函数与拷贝赋值运算符。其实二者函数名完全一样，不一样的是**参数形式从左值引用换成了右值引用**。也因此，一般移动操作都会搭配标准库函数`std::move()`使用，这个函数负责**显式地将一个左值转换为对应的右值类型**，它**返回的是一个右值类型的值**。关于右值引用与`std::move()`的使用我在[c++中的引用](https://xhy3054.github.io/reference-cpp/)这篇博客中有更系统的提到。

如下是对一个对象同时定义拷贝与移动操作：
```cpp
#include <iostream>
#include <string>
#include <utility>
using namespace std;
class HasPtr
{
public:
    HasPtr(const string &s = string()) : ps(new std::string(s)) {cout<<"调用HasPtr()"<<endl;}
    //拷贝
    HasPtr(const HasPtr& hp) :ps(new std::string(*hp.ps)) {cout<<"调用HasPtr(HasPtr&)"<<endl;}

    HasPtr& operator=(const HasPtr& hp){
        cout<<"调用operator=(const HasPtr&)"<<endl;
        auto new_ptr = new std::string(*hp.ps); //防止对自身进行赋值，不能先释放ps指向的内存
        delete ps;
        ps = new_ptr;
        return *this;
    }

    //移动
    HasPtr(HasPtr&& hp) noexcept  //移动过程中不应该抛出异常
    :ps(hp.ps) {
        cout<<"调用HasPtr(HasPtr&&)"<<endl;
        hp.ps = nullptr;    //这样可以使得hp被移动后依然可以正常析构
    }

    HasPtr& operator=(HasPtr&& hp) noexcept    
    {
        cout<<"调用operator=(HasPtr&&)"<<endl;
        //直接检测自赋值
        if(this != &hp){
            cout<<"不是自赋值"<<endl;
            ps = hp.ps;                 
            hp.ps = nullptr;    //使得hp可以正常析构
        }
        return *this;
    }

    ~HasPtr(){delete ps;}

private:
    std::string *ps;
};

int main(){
    HasPtr a;
    HasPtr b(a);
    HasPtr c(std::move(a));
    b = std::move(b);
    a = std::move(c);
    //HasPtr d(a);
    return 0;
}
```
***
如上代码输出结果为：
```bash
xhy@ubuntu:~/cpp_learn/move$ ./move_copy 
调用HasPtr()
调用HasPtr(HasPtr&)
调用HasPtr(HasPtr&&)
调用operator=(HasPtr&&)　
调用operator=(HasPtr&&)　
不是自赋值　
```
---
我们会发现我们**将移动操作声明成`noexcept`**，这个是因为移动操作通常只是**窃取**资源，本身并不分配资源，所以通常不会抛出任何异常。我们提前告诉编译器，可以使编译器可能会为了检测异常做一些额外的工作。

同时移动操作需要**使得自己的参数对象（源目标对象）在被窃取了资源后依然可以正常赋值、析构**。这个操作必须是移动操作来完成，`std::move()`函数只是返回一个右值来表示源对象。使得移动操作可以顺利进行。

## 合成的移动操作
与拷贝操作相同，默认情况下，编译器也会**自动生成移动操作**。但是与默认生成拷贝操作的规则大不相同。

> 通常只有在一个类**没有定义任何自己版本的拷贝控制成员**(包括、拷贝构造、拷贝赋值、析构三种)的情况下，并且此类中的**所有数据成员都支持移动操作**时，编译器才会自动合成移动操作。

## 如果没有移动操作，右值也可以被拷贝
上文中已经提过，也是因为上述拷贝与移动操作的差别，有一些特殊的对象（IO类对象、unique_ptr类对象等）**只支持移动操作而不支持拷贝操作**，比如标准输入输出对象cout与cin，这是因为输入/输出缓冲区只有一个，不可能有多个对象都管理它。此种情况下，在调用移动操作时**必须传入右值，传入左值将会报错**。

同时，更常见的情况是一个类**只支持拷贝操作，不支持移动操作**，此时，**传入右值依旧可以完成拷贝操作**。

> **原因**很好理解，我们在定义拷贝操作时，因为不会对源目标对象做更改，所以常将参数声明成`const &`的类型，而这种类型是可以绑定到一个右值上的。而右值引用不能绑定到一个左值上。





