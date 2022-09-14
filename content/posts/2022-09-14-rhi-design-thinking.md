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

rhi为引擎渲染模块的最底层部分，用来封装不同的渲染api接口，如dx11、dx12、vulkan、mental等；

这里记录的一种设计方法来自引擎[The Forge](https://github.com/ConfettiFX/The-Forge)，基本思路为：

1. 枚举类型的封装，重新**定义一套通用并囊括所有枚举类型的新枚举体**；
2. 资源类型进行封装，**定义通用资源类型**，类里面包含不同api下的资源类型，并以union形式来公用这些api资源类型；
3. 使用前面定义的通用资源类型，**定义渲染流程所需要的通用的函数**，然后在api的cpp文件中去实现当前api的功能实现；在运行时根据所要使用的api，选择对应实现的函数赋给通用函数使用即可；

后面要看的rhi设计有[nvrhi](https://github.com/NVIDIAGameWorks/nvrhi)，filament，unreal，cocos3d；