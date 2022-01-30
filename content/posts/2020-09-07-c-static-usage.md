---
title: 'C++ Static Usage'
date: 2020-09-07 14:37:51
categories:
- C++
tags: 
- C++
---

C++中的static关键字的一些使用细节问题
<!--more-->

## C语言中的static关键字

### 修饰全局变量，全局函数

将限制该变量及函数的作用域为本文，不能实现连接时的跨文本使用；

```C++
//file1.c
static int a = 10;      //变量作用范围限制在本文本作用域中

//file2.c
#include <iostream>

using namespace std;

extern int a;   //无法使用file1.c中的a变量

int main()
{
    cout << "a = " << a <<endl;

    return 0;
}

```

### 修饰局部变量

在用static修饰局部变量后，该变量只在初次运行时进行初始化工作，且只进行一次；且该变量便存放在静态数据区，其生命周期一直持续到整个程序执行结束。

```c++
#include<stdio.h>

void fun()
{
    static int a=1;
    a++;
}

int main(void)
{
    fun();  //这里运行时，a会进行初始化，随后a++
    fun();  //这里运行时，只会运行a++
    return 0;
}
```
