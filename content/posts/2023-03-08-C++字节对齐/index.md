---
title: 'C++字节对齐'
author: wingstone
date: 2023-03-08
categories:
- C++
tags: 
- C++
---

记录C++字节对齐的标准，避免老是忘记；

<!--more-->

## 为何需要对齐

1. 为了运行效率；电脑处理器读取内存是以2，4，8等2的幂次方的字节chunk来进行读取的，假如数据没有进行字节对齐，处理器可能就需要进行两次内存读取才能还原原来的数据，从而大大影响效率；在某些x86的某些设置下，或某些处理器下，甚至会直接触发指令错误；
2. 满足原子操作的条件：假如数据没有对齐，需要两次内存读取，那么就可能遇到两个内存分别处在不同的虚拟内存页上，假如一个内存页没有加载，就会触发页面中断，再运行页面加载指令，从而打断了原子操作的要求；
3. 缓存友好：某些CPU的cacheline为64byte，满足字节对齐可以更容易塞满缓存，充分利用缓存大小；

基于此，各大处理器架构提出了相似的对齐要求，用行话来说来说就是自对齐，char数据可以是任意字节地址，2 byte short数据地址必须能被2整除，4 byte int数据地址必须能被4整除，32 bit指针要进行4byte对齐，64 bit指针要进行8byte对齐；

## 编程中的对齐

幸运的是，编译器会自动帮我们进行处理，来使得以上对齐规则得以满足；

 **1. 针对包含不同数据类型的结构体，编译器可能会主动在数据之间插入padding，来使得结构体内部的各个数据类型能够满足对齐要求；** 如下：

```c++
struct Data
{
    char a;             // 占1个字节+3字节padding，编译时，添加3个字节的padding，来使得b能够满足4字节的int对齐；
    int b;              // 占4个字节，编译时，不需要进行额外处理，当前4个字节加上前面的4个字节，刚好能使得c能够满足8字节的double对齐；
    double c;           // 占8个字节, 不需要额外处理；
}
```

 **2. 除了保证结构体内部的数据类型对齐，对于结构体本身也有对齐要求，就是结构体字节大小需要为结构体内部最大数据类型字节数的倍数，该对齐要求能够保证申请结构体数组时，数组内各结构体的各个数据成员都能满足上面提到的数据对齐要求；** 如下：

```c++
struct Data
{
    double a;           // 占8个字节；
    char b              // 占1个字节+7个字节padding，编译时添加7个字节的padding，来使得结构体字节大小最大数据类型double字节数的倍数；
}
```

其实道理也很简单，我们申请一个数组为Data data[2]，展开后内部的数据分布为

```c++
{
    double a0,
    char b0,
    double a1,
    char b1,
}
```

假如没有padding，数据a1就不能满足数据对齐要求，从而导致各种延伸问题；

## 编程中面对对齐应该如何做

知道了对齐规则，我们在编程中应该如何操作才能更好的利用对齐；注意点有两点：

 **1. 注意结构体内部数据的排序，使得结构体大小最小，从而能够节省系统内存；** 例如：

```c++
struct Data1
{
    char a;             // 1字节+7字节padding
    double b;           // 8字节
    int c;              // 4字节+4字节padding
}
struct Data2
{
    char a;             // 1字节+3字节padding
    int b;              // 4字节
    double c;           // 8字节
}
```

以上两个结构体，内数存储数据是一样的，但是Data2使用了更好的排序方式，从而使得结构体占用内存更小；

 **2. 如果结构体中在编译时会添加padding，不如在申明时手动添加padding，从而能使代码更加清晰；** 例如：

```c++
struct Data1
{
    double a;               // 8字节
    int b;                  // 4字节+4字节padding
}
struct Data2
{
    double a;               // 8字节
    int b;                  // 4字节
    int padding;            // 4字节
}
```

以上两个结构体，所占用字节大小是一样的，但是Data2使用了显式的padding，从而使得代码使用者能够一眼看清内部数据成员数量，一个所占用字节大小；

关于更复杂情况下的字节占用计算，推荐阅读[类大小计算](https://github.com/Light-City/CPlusPlusThings/tree/master/basic_content/sizeof)；

## Reference

1. [Byte alignment and ordering](https://eventhelix.com/embedded/byte-alignment-and-ordering/#:~:text=General%20Byte%20Alignment%20Rules%201%20Single%20byte%20numbers,%20total%20structure%20is%204%20bytes.%20More%20items)
2. [Purpose of memory alignment](https://stackoverflow.com/questions/381244/purpose-of-memory-alignment)
3. [Data alignment: Straighten up and fly right](https://developer.ibm.com/articles/pa-dalign/)
4. [The Lost Art of Structure Packing](http://www.catb.org/esr/structure-packing/)