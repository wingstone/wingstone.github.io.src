---
title: 'rhi design thinking'
author: wingstone
date: 2022-09-14
categories:
- contents
metaAlignment: center
coverMeta: out
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
2. 资源类型进行封装，**定义通用资源类型**，这里的通用资源类型为抽象类，内部并不含有实际资源类型；不同api进行该类型的继承，并在子类中存储该api下实际的资源类型；
3. 使用前面定义的通用资源类型，**定义渲染流程所需要的通用抽象函数**，一般这部分抽象函数定义在抽象device中，各api继承该device类，并使用前面的通用资源类型指针（利用资源类型的运行时多态），进行抽象函数的真正实现；

> 总来的来说，基本上是使用C++面向对象风格的rhi抽象；

### unreal

[unreal rhi](https://github.com/EpicGames/UnrealEngine)的设计与nvrhi思路基本一致，知乎上有一篇[文章](https://zhuanlan.zhihu.com/p/417561163)对此分析的很透彻，可以进行参考；阅读时所要查看的相关文件有：

1. 整体rhi抽象的定义位于**Engine\Source\Runtime\RHI**与**Engine\Source\Runtime\RHICore**；
2. 各个平台api对应的实现分别位于**Engine\Source\Runtime\D3D12RHI**、**Engine\Source\Runtime\VulkanRHI**、**Engine\Source\Runtime\Windows\D3D11RHI**、**Engine\Source\Runtime\Apple\MetalRHI**、**Engine\Source\Runtime\OpenGLDrv**;
3. 其中，资源类型抽象类的封装位于RHIResources.h；
4. 其中，抽象函数的封装位于RHIContext.h；

值得注意的是，unreal单独做了一个rhi thread来负责api指令的分发，一个是可以利用新一代图形api支持多线程的特性，来充分利用多线程的优势；二是rhi thread执行时不会阻塞game thread的执行，从而提升运行效率，还能提升帧率的稳定性；

### filament

[filament](https://github.com/google/filament)中的rhi设计在与nvrhi以及unreal也基本一致；

1. 所有的抽象层都位于**filament\backend**路径下；
2. DriverBase.h文件中定义了所有的抽象层资源；
3. DriverAPI.inc文件，以纯字符串的形式定义了所有的抽象层api函数；
4. Driver.h文件定义了driver抽线层，以include的形式内置了DriverAPI.inc所包含的抽象api函数；
5. **filament\backend\src\vulkan**、**filament\backend\src\opengl**、**filament\backend\src\metal**分别实现了各平台下的driver，通过继承Driver.h中的DriverBase来实现，同样以include的形式内置了DriverAPI.inc所包含的抽象api函数；