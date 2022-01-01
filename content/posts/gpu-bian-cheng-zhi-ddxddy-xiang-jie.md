---
title: 'GPU编程之DDX、DDY详解'
date: 2020-09-21 16:18:43
tags: [GPU,图形学]
published: true
hideInList: false
feature: 
isTop: false
---
ddx、ddy函数的含义为返回大约目前变量的偏导数；

所谓偏导数，实际上就是分别在x方向和y方向的单位距离下，变量的差值；

在GPU中用数值进行计算的话，就是相邻像素下该变量的差值；

## 使用细节

对于矢量变量，仍然返回一个矢量，但是矢量中的每个元素都会计算其偏导数；

该函数只能在fragment program profiles中使用，但是并不是所有的硬件都支持；

当该函数使用在条件分支语句中，ddx、ddy的偏导计算并不是完全正确的，因为并不是所有的fragment都运行在同一个分支中；

当变量所依据的参数是基于中心插值的情况下，偏导计算并不是非常精确；

## 实现细节

其偏导数的实现方式要依据具体的实现方式；典型的实现方式为fragment被光栅化时，会以为2x2的形式形成像素块（称作quad-fragments），偏导计算就在quad-fragments中相邻的fragment中进行差值计算；这是像素着色器上面的最小工作单元 ，在像素着色器中，会将相邻的四个像素作为不可分隔的一组，送入同一个SM内4个不同的Core。

[NV shader thread group](https://www.khronos.org/registry/OpenGL/extensions/NV/NV_shader_thread_group.txt)提供了OpenGL的扩展，可以查询GPU线程、Core、SM、Warp等硬件相关的属性。在处理2x2的像素块时，那些未被图元覆盖的像素着色器线程将被标记为`gl_HelperThreadNV = true`，它们的结果将被忽略，也不会被存储，但仍然进行着色器的计算，可辅助一些计算，如导数`ddx`和`ddy`。

原文：
> The variable gl_HelperThreadNV specifies if the current thread is a helper thread. In implementations supporting this extension, fragment shader invocations may be arranged in SIMD thread groups of 2x2 fragments called "quad". When a fragment shader instruction is executed on a quad, it's possible that some fragments within the quad will execute the instruction even if they are not covered by the primitive. Those threads are called helper threads. Their outputs will be discarded and they will not execute global store functions, but the intermediate values they compute can still be used by thread group sharing functions or by fragment derivative functions like dFdx and dFdy.

## Reference

[cg-ddx](https://developer.download.nvidia.cn/cg/ddx.html)
[深入GPU硬件架构及运行机制](https://www.cnblogs.com/timlly/p/11471507.html#436-%E5%83%8F%E7%B4%A0%E5%9D%97%EF%BC%88pixel-quad%EF%BC%89)