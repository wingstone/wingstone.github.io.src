---
title: Copy control in c++
author: wingstone
date: 2022-05-09
categories:
- C++
tags:
- 心得
metaAlignment: center
coverMeta: out
---

Copy control在c++中及其重要，使用不当，会严重影响c++的运行效率；本文总结主要参考C++ primer的第13章节，再加上一些个人的理解；
<!--more-->

## Copy Constructor

Copy构造函数以相同类型实例的引用来作为第一个参数，剩余的参数没有或者使用默认参数即可；如下代码所示：
```C++
class Foo {
public:
    Foo(); // default constructor
    Foo(const Foo&); // copy constructor
    // ...
};
```

Copy构造函数的第一个参数几乎都是const的，虽然可以不加const，但是从代码的安全性考虑，不建议这样做；

Copy构造函数被调用的情况包含Copy Initialization，以及函数的参数，返回值；这些都是Copy构造函数的implicit调用（由编译器生成）；除了implicit调用，还有匹配Copy构造函数声明的explicit调用，如：
```c++
#include <iostream>
class Something
{
public:
	Something() = default;
	Something(const Something&)
	{
		std::cout << "Copy constructor called\n";
	}
};

Something goo(Something other)          // copy constructor for Parameters; implicit
{
	Something s = other;                // copy constructor for Parameters; implicit
	return s;                           // copy constructor for return values; implicit
}

int main()
{
	std::cout << "Initializing s1\n";
	Something s0;
	Something s1 = goo(s0);             // copy constructor for copy initialization; implicit
	Something s2(s1);                   // copy constructor for copy initialization; explicit

	return 0;
}
```

如果声明Copy constructor时添加了explicit声明，将只能进行Copy constructor的显式调用，不能进行隐式调用；如：
```c++
#include <iostream>
class Something
{
public:
	Something() = default;
	explicit Something(const Something&)
	{
		std::cout << "Copy constructor called\n";
	}
};
int main()
{
	std::cout << "Initializing s1\n";
	Something s0;
	Something s2 = Something();         // error! copy constructor for copy initialization; implicit!
	Something s1(s0);                   // copy constructor for copy initialization; explicit

	return 0;
}
```

事实上，编译器在一些情况下会对copy initialization进行省略优化，从而转化为调用对应的direct initilization；
以下例子在VS2019上运行，实际上只有一次copy initialization的调用；
```c++
#include <iostream>
class Something
{
public:
	Something() = default;
	Something(const Something&)
	{
		std::cout << "Copy constructor called\n";
	}
};

Something foo()
{
	return Something(); // copy constructor normally called here
}
Something goo()
{
	Something s;
	return s; // copy constructor normally called here
}

int main()
{
	std::cout << "Initializing s1\n";
	Something s1 = foo(); // copy constructor normally called here

	std::cout << "Initializing s2\n";
	Something s2 = goo(); // copy constructor normally called here
}
```

实际上，在STL的container的push_back与emplace_back的实现中，push_back使用copy initialization来填充，以对象为参数，emplace_back使用direct initialization来填充，以构造函数参数为参数，并返回新构造的对象；

## Copy-Assignment Operator

Copy赋值操作与Copy构造函数的区别在于，Copy赋值操作只有explicit调用，并且调用实际与Copy构造函数不同，如下：
```c++
#include <iostream>
class Something
{
public:
	Something() = default;
	Something(const Something&)
	{
		std::cout << "Copy constructor called\n";
	}
	Something& operator = (const Something&)
	{
		std::cout << "Copy Assignment called\n";
		return *this;
	}
};

int main()
{
	Something s1;
	Something s2 = s1;		// copy constructor normally called here

	Something s3;
	s3 = s2;				// copy Assignment normally called here
}
```

## = default and = delete

Copy赋值操作与Copy构造函数，如果没有编写，编译器会在内部生成对应的函数实现；但这样会导致这些函数对使用者呈现隐藏状态，对使用者不够友好；
因此在C++11中，使用=default，表示明确告诉编译器，使用编译器的默认实现；使用=delete，表示明确告诉编译器，不需要进行相应实现；
若=default在函数声明处添加，表示为inline函数；若=default在函数实现处添加，表示为非inline函数；
```c++
class Sales_data {
public:
    Sales_data() = delete;                      // no implement
    Sales_data(const Sales_data&) = default;    // default implement, inline
    Sales_data& operator=(const Sales_data &);
    ~Sales_data() = default;                    // default implement, inline
};
    Sales_data& Sales_data::operator=(const Sales_data&) = default;     // default implement, not inline
```

## Rvalues

右值与左值相对应，区分的方法为，可以放到=左边的值就是左值，不能放到=左边的就是右值；
更通俗的区分方法为，我们使用右值，使用的是右值的内容，使用左值，使用的是左值的地址；
如：
```c++
int a = 1;  // a为左值，a可以放在=左边，a可以取地址；1为右值，1不能放在=左边，1不能取地址，只能取内容；
a = a + 1;  // a为左值，a + 1为右值；
a = abs(a, 1);  // a为左值，abs(a, 1)为右值；
```

## Rvalue References

右值虽不能取地址，但是在C++11中可以取右值的引用，使用方法为声明右值引用时，添加&&来表示；
值得注意的是右值引用只能绑定右值，不能绑定其它；
如：
```c++
int i = 42;
int &r = i;                 // ok: r refers to i
int &&rr = i;               // error: cannot bind an rvalue reference to an lvalue
int &r2 = i * 42;           // error: i * 42 is an rvalue
const int &r3 = i * 42;     // ok: we can bind a reference to const to an rvalue
int &&rr2 = i * 42;         // ok: bind rr2 to the result of the multiplication
```

### Lvalues Persist; Rvalues Are Ephemeral

左值可以长期持有，而右值通常是一些将要销毁的值，因为没有其它值来使用这些右值；因此右值引用就变成了用来获取右值所占有资源的唯一途径，我们可以通过后续的移动构造或赋值来使用这些资源；

### Variables Are Lvalues

值得注意的是右值引用本省是个变量，而变量是可以放到=左边的，因此右值引用本身是个左值；
```c++
int &&rr1 = 42;         // ok: literals are rvalues
int &&rr2 = rr1;        // error: the expression rr1 is an lvalue!
```

### The Library move Function

虽然右值引用不能绑定左值，但是还是可以通过static_cast来将左值cast为右值，从而可绑定到右值引用上；
在C++标准库里面有一个move函数，其内部的实现既是通过cast来完成的；
```c++
int rr1 = 1;
int &&rr3 = std::move(rr1);     // ok
```

## Move Constructor and Move Assignment

所谓移动构造函数与移动赋值函数，与拷贝构造函数类似，只不过移动函数使用右值引用来作为参数，并且实现上，挪用右值的资源为己用，而不是拷贝重新构造一份；
对于，io buffer以及unique_ptr这些对象不能进行共享，因此对应的资源也非常适合移动，而不是拷贝；

## Move Operations, Library Containers, and Exceptions

```c++
class StrVec {
public:
    StrVec(StrVec&&) noexcept; // move constructor
    // other members as before
};
StrVec::StrVec(StrVec &&s) noexcept : /* member initializers */
{ /* constructor body */ }
```
我们必须在move函数的声明与定义处，添加no exception标签；
原因1：我们的move函数只挪用资源，不分配与释放资源，因此没有异常弹出，显式告诉编译器，可以减少编译器部分工作；
原因2：对于container，在其push_back时，其保证了有exception时，原对象是不变的，因此如果有exception，container就会调用拷贝构造函数，来保证原对象不变；使用noexcaption可以保证container调用move构造函数；

## Move-Assignment Operator

Move赋值操作所执行内容与Move constructor基本一致，同时又与Copy赋值操作一样，函数最终要返回自身的引用；如：
```c++
StrVec &StrVec::operator=(StrVec &&rhs) noexcept
{
    // direct test for self-assignment
    if (this != &rhs) {
        free();                         // free existing elements
        elements = rhs.elements;        // take over resources from rhs
        first_free = rhs.first_free;
        cap = rhs.cap;
                                        // leave rhs in a destructible state
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}
```

## A Moved-from Object Must Be Destructible

被move后的对象，必须要保证其是可销毁的，因为其内部资源已被move，那么被move后的对象就处于一个随时可被销毁的状态；上面例子的`rhs.elements = rhs.first_free = rhs.cap = nullptr;`就保证了被move后的对象可被安全销毁；

## The Synthesized Move Operations

编译器也可以合成对应的Move函数，条件为：
1. Unlike the copy constructor, the move constructor is defined as deleted if the class has a member that defines its own copy constructor but does not also define a move constructor, or if the class has a member that doesn’t define its own copy operations and for which the compiler is unable to synthesize a move constructor. Similarly for move-assignment.
2. The move constructor or move-assignment operator is defined as deleted if the class has a member whose own move constructor or move-assignment operator is deleted or inaccessible.
3. Like the copy constructor, the move constructor is defined as deleted if the destructor is deleted or inaccessible.
4. Like the copy-assignment operator, the move-assignment operator is defined as deleted if the class has a const or reference member.

## Rvalues Are Moved, Lvalues Are Copied

如果copy函数与move函数同时存在，则按照函数参数来匹配对应的函数调用，对constructor与assignment是一致的；
如果不存在Move函数，则会调用对应的copy函数；

## Reference

1. [Is there a difference between copy initialization and direct initialization?](https://stackoverflow.com/questions/1051379/is-there-a-difference-between-copy-initialization-and-direct-initialization)
2. [Copy initialization](https://www.learncpp.com/cpp-tutorial/copy-initialization/)
3. [c++为什么要搞个引用岀来，特别是右值引用，感觉破坏了语法的简洁和条理，拷贝一个指针不是很好吗？](https://www.zhihu.com/question/363686723)
4. [在拥挤和变化的世界中茁壮成长：C++ 2006–2020](https://github.com/Cpp-Club/Cxx_HOPL4_zh)