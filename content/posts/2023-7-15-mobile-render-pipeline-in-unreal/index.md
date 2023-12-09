---
title: 'mobile render pipeline in unreal'
author: wingstone
date: 2023-07-15
categories:
- unreal
tags: 
- pbr
---

归纳总结unreal中移动渲染管线的主要顺序及思路，以及这样做背后的思考，便于以后修改ue移动管线时有一个全局的把控；

<!--more-->

## 前置理论知识

理解unreal的多线程渲染架构，其中game线程与render线程的class对应关系需要牢记在心里，参考[
Graphics Programming Overview](https://docs.unrealengine.com/5.1/en-US/graphics-programming-overview-for-unreal-engine/)；如下所示：

|Game Thread | Rendering Thread|
|---|---|
|UWorld | FScene|
|UPrimitiveComponent | FPrimitiveSceneProxy / FPrimitiveSceneInfo|
| | FSceneView / FViewInfo|
|ULocalPlayer | FSceneViewState|
|ULightComponent | FLightSceneProxy / FLightSceneInfo|

## unreal如何调用一个管线







## Reference

1. [Byte alignment and ordering](https://eventhelix.com/embedded/byte-alignment-and-ordering/#:~:text=General%20Byte%20Alignment%20Rules%201%20Single%20byte%20numbers,%20total%20structure%20is%204%20bytes.%20More%20items)
2. [Purpose of memory alignment](https://stackoverflow.com/questions/381244/purpose-of-memory-alignment)
3. [Data alignment: Straighten up and fly right](https://developer.ibm.com/articles/pa-dalign/)
4. [The Lost Art of Structure Packing](http://www.catb.org/esr/structure-packing/)