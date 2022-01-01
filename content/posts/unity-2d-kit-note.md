---
title: ' Unity 2D Kit Note'
date: 2020-03-29 14:32:39
tags: [unity]
published: true
hideInList: false
feature: 
isTop: false
---

介绍unity 2D Kit包中所用的一些组件；

## Sorting Group组件

Sorting Group组件可以改变Sprite Renderers的渲染顺序，Sorting Group作为Sprite Renderers的覆盖层，用于区分Group之间的渲染顺序；

1. Sorting Layer可以改变物体以及子物体的排列层，不同的排列层之间有渲染顺序区分；
2. Order In Layer可以改变物体及子物体在排列层内的渲染顺序；

## Sprite Renderer组件

渲染sprite时，使用的mesh不一定是一个四边形，其形状以能拟合Sprite的大致形状为准；

1. Flip可以控制X或Y方向的翻转

2. Draw Mode控制Sprite Renderer尺寸修改情况下的绘制方式；

3. Mask Interaction控制Sprite Mask存在时，Sprite Render与mask之间的显示方式；

4. Sprite Sort Point用来控制sprite Render绘制顺序的控制点，来计算其到相机的距离，优先级低于Sorting Layer、Order In Layer；

## StartScreenSpriteOffsetter组件

用于图片随鼠标偏移，可改进为随重力偏移

## NormalMappedTileAnimated材质

使用SineApproximation来模拟sin函数的顶点动画surface shader，没想到其着色方式竟然是完整的PBR==

## Scene Controller组件

场景控制的单例模式脚本，用来控制场景的切换

## BackgroundMusicPlayer组件

背景音乐控制的单例脚本，用于播放背景音乐的控制

## ScreenFader组件

控制屏幕渐变的单例组件，用于全屏渐变效果

## Platform Effector 2D组件

用来做横版2D游戏从下往上跳而不被阻挡的那种跳跃平台用的。[还是中文读起来舒服](https://www.cnblogs.com/weiqiangwaideshijie/p/7007256.html);

## TileMap组件

搭建2D场景用的网格用组件
