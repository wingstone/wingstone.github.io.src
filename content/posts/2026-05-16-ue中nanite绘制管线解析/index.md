---
title: 'ue中nanite绘制管线解析'
author: wingstone
date: 2026-05-16
categories:
- UE
- Nanite
tags: 
- UE
---

简易分析UE内部nanite绘制管线实现，需要对nanite有基本的了解；

<!--more-->

## 前置理论知识

Nanite管线的大致理解，可参考[UE5渲染技术简介：Nanite篇](https://zhuanlan.zhihu.com/p/382687738)。简单来说就是，运用GPU Driven Culling + Soft Restarization + Visibility Buffer的混合技术来高效渲染高密度模型，提高大量三角形下的渲染效率。

GPU Driven Culling用来解决三角形overdraw问题，但剔除粒度在instance与meshlet级别；

Soft Restarization用来解决quad overdraw问题，三角形的硬件光栅化会以2x2的quad为基本单位，小于2x2的三角形会造成overdraw，该技术就是为了解决此问题；

Visibility Buffer用来解决带宽与pixel overdraw问题，若没有vbuffer，需要使用gbuffer作为对应rt，使用vbuffer可以减少gbuffer的读写带宽开销，同时vbuffer能省略不可见像素的texture写入gbuffer部分，在减小带宽的同时还能移除pixel overdraw问题；

整体流程为：

![Nanite Pipline](image.png)

## GPU Driven Culling

这部分特殊的地方在于Nanite采用了Two Pass Occlusion Culling的方式来进行instance与meshlet的剔除；具体可参考[理解Nanite（一）：遮挡剔除](https://zhuanlan.zhihu.com/p/583245401)；

剔除主要采用当前视锥体剔除，与上一帧的hzb剔除，包括instance cull与meshlet cull；

在main pass中会将当前instance、meshlet的bound转换到上一帧，然后用上一帧的hzb剔除不可见的instance、meshlet；为什么需要将bound转换到上一帧，而不是将上一帧的hzb转换到当前帧？因为如果将上一帧的hzb转换到当前帧，像素之间可能会出现不连续性，从而出现裂缝，影响剔除；

剔除过后会使用留下的instance、meshlet进行光栅化获取vbuffer，随后使用vbuffer来构建当前帧的hzb；

在post pass使用当前帧构建的hzb对main pass剔除过的instance、meshlet再进行保守剔除，对未剔除的instance、meshlet再进行光栅化获取最终vbuffer，随后再构建当前帧最终的hzb；


## Restarization

光栅化分为采用cs的soft restarization与采用ps的hard restarization；在main pass与post pass中均会使用到restarization，且流程基本是一致的；

具体实现可NaniteCullRaster.cpp文件中的FRenderer::AddPass_Rasterize函数；

总体来说，restarization的流程首先将cluster进行分类，根据cluster的屏幕占比，来决定是使用soft restarization还是hard restarization，该流程称之为Raster Binning；随后分别进行soft restarization或hard restarization；

其中soft rastazation的实现是通过cs来实现的，在cs中进行cluster内三角形的插值，并根据最新所支持的64位图像原子操作，来写入里屏幕最近的深度；

HW Rasterize的实现会根据选择来使用不同的硬件方式，分别是mesh shader以及vertex shader；

## Visibility Buffer

光栅化后获取的是一个64位的vbuffer，该vbuffer中存储了triangle id、cluster id与depth信息；

基于vbuffer中的triangle id与cluster id，可以获取到cluster所包含的顶点信息，进而可以将triangle的三个顶点坐标进行插值，从而还原出triangle在screen space中的位置，进而可以对其他顶点属性进行插值操作，可以使用插值后的uv进行进行贴图采样；

为了与ue之前的延迟管线相结合，ue并没有直接使用vbuffer进行光照计算，而是将vbuffer中的信息转换成了gbuffer格式，后续的光照计算还是基于gbuffer进行；