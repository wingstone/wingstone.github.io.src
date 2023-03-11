---
title: 'sampler in path tracing'
author: wingstone
date: 2022-12-22
categories:
- 离线渲染
- path tracing
- sampling theory
tags: 
- 总结
- path tracing
---

这篇文章主要记录学习离线渲染中与采样相关的理论与实践；

<!--more-->

## The Halton Sampler

halton sampler基本上为最常见的低差异序列生成方法；

而在实际的离线rendering过程，有些采样是处在同一维度的，有些采样是跨维度的，需要进行合适的区分，否则就使用了错误的地差异序列；

像素内的射线起点的多采样，为两个维度下halton采样；一般会直接预生成二维下对应采样数的采样点，然后直接使用；

tracing过程中，每次计算着色点，采样光源，采样brdf，都属于不同的维度（可能是一维也可能是二维），需要使用不同维度下halton采样；即我们常用的`Get1D()`或`Get2D()`；