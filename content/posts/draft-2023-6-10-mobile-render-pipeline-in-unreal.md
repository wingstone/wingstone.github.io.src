---
title: 'mobile render pipeline in unreal'
author: wingstone
date: 2023-07-15
categories:
- unreal
tags: 
- renderpipeline
draft: true
---

归纳总结unreal中移动渲染管线的主要顺序及思路，以及这样做背后的思考，便于以后修改ue移动管线时有一个全局的把控；

<!--more-->

## 前置理论知识

理解unreal的多线程渲染架构，其中game线程与render线程的class对应关系需要牢记心里，参考[Graphics Programming Overview](https://docs.unrealengine.com/5.1/en-US/graphics-programming-overview-for-unreal-engine/)；如下所示：

|Game Thread | Rendering Thread|
|---|---|
|UWorld | FScene|
|UPrimitiveComponent | FPrimitiveSceneProxy / FPrimitiveSceneInfo|
| | FSceneView / FViewInfo|
|ULocalPlayer | FSceneViewState|
|ULightComponent | FLightSceneProxy / FLightSceneInfo|

理解unreal的FrameGraph框架，参考[Render Dependency Graph](https://docs.unrealengine.com/5.1/en-US/render-dependency-graph-in-unreal-engine/)；

## unreal如何调用一个管线

