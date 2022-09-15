---
title: 'rhi design thinking'
author: wingstone
date: 2022-09-4
categories:
- contents
metaAlignment: center
coverMeta: out
draft: true
---

rhi设计的心得记录

<!--more-->

rhi为引擎渲染模块的最底层部分，用来封装不同的渲染api接口，如dx11、dx12、vulkan、mental等；这里记录一些不同引擎的rhi设计方法；

### The Forge
第一个设计方法来自引擎[The Forge](https://github.com/ConfettiFX/The-Forge)，里面有visibility buffer的开源实现，基本思路为：

1. 枚举类型的封装，重新**定义一套通用并囊括所有枚举类型的新枚举体**；
2. 资源类型进行封装，**定义通用资源类型**，类里面包含不同api下的资源类型，并以union形式来公用这些api资源类型；
3. 使用前面定义的通用资源类型，**定义渲染流程所需要的通用的函数指针**，然后在各api的cpp文件中去进行当前api的功能实现；
4. 在运行时根据所要使用的api，将对应实现的函数地址赋给通用函数指针，使用通用函数指针进行管线搭建即可；

> 总来的来说，基本上是使用C风格的rhi抽象；

### nvrhi

第二个设计方法来自nvidia所开源的[nvrhi](https://github.com/NVIDIAGameWorks/nvrhi)，nvidia的很多图形demo的底层都是使用的此库；基本思路为：

1. 枚举类型的封装，重新**定义一套通用并囊括所有枚举类型的新枚举体**；
2. 资源类型进行封装，**定义通用资源类型**，这里的通用资源类型为抽象类，内部并不含有实际资源类型；不同api进行该类型的继承，并在子类中存储改api下实际的资源类型；
3. 使用前面定义的通用资源类型，**定义渲染流程所需要的通用抽象函数**，一般这部分抽象函数定义在抽象device中，各api继承改device类，并使用前面的通用资源类型指针（利用资源类型的运行时多态），进行抽象函数的真正实现；

> 总来的来说，基本上是使用C++面向对象风格的rhi抽象；

后面要看的rhi设计有filament，unreal，cocos3d；