---
title: '软阴影技术——PCF、ESM、VSM、CSM、PCSS'
date: 2020-09-30 23:39:42
tags: []
published: true
hideInList: false
feature: 
isTop: false
---

软阴影技术的各种实现以及个人理解；
<!--more-->

## PCF（Percentage-Closer Filtering）

就是对阴影结果进行滤波，这里的阴影结果指阴影测试函数所得的结果；一般采用双线性滤波，此时可以使用硬件PCF来进行插值；

### 实现要求

Depth格式的shadow map或者ShadowMap格式的shadow map；

## ESM（Exponential Shadow Maps）

PCF方法对阴影结果进行滤波，无法集成到阴影测试函数中；因此可以采用其他的阴影测试函数来进行实现软阴影效果；

ESM采用指数空间下的深度测试函数来代替传统的深度测试函数；

采用ESM方法，存储的为指数空间下的阴影数值，支持预滤波，这样就可以对shadowmap进行blur，将滤波与测试函数进行分离；

传统shadowmap存储的为深度z，而在ESM中存储的为`exp(c*z)`，即深度值的指数形式；其中c表示指数常数，对于32位存储格式，极限值为88；

而阴影测试函数，传统的shadowmap为`step(d, z)`，而ESM的阴影测试函数则变为`exp(-cd)*tz`；其中tz表示采样的指数深度值，即`exp(c*z)`，这样原来的`z-d`转变为了`exp(c(z-d))`；

### 所带来的问题有：

1. 计算出来的阴影值与shadow caseter、shadow receiver之间的距离有关，距离越远，阴影越黑，越近，阴影越接近于无；
2. 多重阴影下，肯会由于shadow map精度问题产生瑕疵；
3. 对rt精度要求比较高；

### 实现需求

一张Depth格式的shadow map即可，需要手动对其进行模糊；

## Renference

1. 《实时阴影技术》 艾森曼努；
2. [实时渲染中的软阴影技术](https://zhuanlan.zhihu.com/p/26853641)
3. [切换到esm](http://www.klayge.org/2013/10/07/%e5%88%87%e6%8d%a2%e5%88%b0esm/)