---
title: "Fine Pruned Tiled Light"
author: wingstone
date: 2022-01-04T11:43:02+08:00
categories:
- GPU Pro
tags:
- lighting
- efficient
metaAlignment: center
coverMeta: out
draft: true
---

Fptl是基于tiled base lighting而开发出来的更高效的光照计算方法，相比于tbl的光源剔除粒度（tiled级别下的剔除），Fptl可以达到逐像素级别的剔除，从而使光照计算更加高效。
<!--more-->

## 概述

Fptl已经应用在**Rise of the Tomb Raider**游戏中，Unity的hdrp中也使用了这一技术；在实际过程中，light list的生成会在渲染shadow map时，借助computer shader进行异步计算；

light list的生成会分为两步：
- 第一步生成初步的light list，此时light list的生成，由tile对应的screen space bounding volume与光源对应的aabb进行求交计算得来，计算结果存储在local storage中；
- 第二步进行精细剔除，针对tile逐像素计算世界坐标，然后针对第一步得到的初步light list进行遍历，计算世界坐标是否在light shape范围内，若有像素在light shape内，则对应tile的fine light list应该包含此光源；

## 介绍

传统的延迟渲染使用alphablend来进行光照的叠加，最大的优点是拥有分配和计算像素级别剔除的光照，对应3d空间中真正能被光源照射到的点；

对于前向渲染，light list的构建主要在CPU端，会进行mesh的bounding volume与light volume的求交计算；这种方法会导致mesh会再像素级别上产生over lighting，对于比较大的mesh尤为严重；

由于Dx11的引入，基于cs的tiled lighting成为延迟渲染里主流的选择；TL会将屏幕划分`$n \times m$`的固定分辨率的tile，GPU会对每个tile生成一个index列表，这是一个与当前tile重叠的光源索引；后面的lightpass会根据这个index列表，来加载当前tile计算光照所需要的所有光源；

TL的大致流程：
1. 针对每一个tile，计算改tile深度的最大值与最小值，得到一个tile bound；
2.  Each thread checks a disjoint subset of lights by bounding sphere against tile bounds.
3. 将与该tile的相交的light的indices存储到local data storage（LDS）；
4. 得到的light list则被所有的thread用来进行当前tile的光照运算；

TL的优点：
1. 对于TDL（tiled deferred lighting），针对每个像素使用**一个pass**循环当前tile的light list，即可完成所有的光照；相比于DL（deferred lighting），TDL只需要采样一次G-buffer，写入一次frame buffer即可；
2. TL不仅可以用于DL，也可用于FL（forward lighting）；
3. 对于TL来说，light list的生成非常适合使用异步计算，使得我们可以将这一过程与不相关的渲染计算同时进行；

TL的缺点：TL根据tile bound与light volume相交计算来得到light list；由于light volume（一般用AABB表示）并不能精确地表示光源的范围，因此计算得到的light list其实是有冗余光源的；

TL的解决办法：使用精确的light shape来表示光源的范围，使用使用light volume进行粗剔除，再使用light shape进行精确的剔除，就能得到像素级别精确的light list；

## fine pruned tiled light

在Fptl发明者的项目里面：当前的视锥体里，不会超过40-120盏灯光；经常会出现少量灯光占据大片屏幕面积；经常会使用狭长的spot light来达到比较好的光照分布，使用light volume会产生光照的浪费；

这些情况使得传统的TL不能很好的解决light list生成问题，因此Fptl才被开发出来，大致流程如下：

针对每个相机：
1. 在CPU端计算与相机视锥体有交集的所有光源；
2. 根据光源的类型对光源进行排序；
3. 在GPU端，根据每个light的light shape来计算其紧致screen space AABB；通过求取相机与light的convex hull的insection volume来计算，并进一步限制了bound sphere的AABB；

针对每个16x16tile：
1. 针对每个tile计算depth buffer的min depth与max depth；
2.  Each thread checks a disjoint subset of lights by bounding sphere against tile bounds.
3.  将与该tile的相交的light的indices存储到local data storage（LDS）；可将此indices称之为coarse list；
4.  在同一个kernel下，循环coarse list的所有light；
每一个thread计算4个像素，计算该像素是否在某一light shape内；然后将计算结果存储到一个bit位上，该bit系列对应coarse list，每个bit表示该light是否确实与tile相交；

## 实现细节