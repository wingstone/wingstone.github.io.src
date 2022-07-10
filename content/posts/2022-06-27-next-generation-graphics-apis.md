---
title: An overview of next-generation graphics APIs
author: wingstone
date: 2022-06-27
categories:
- graphics api
tags:
- vulkan
- mental
- dx12
metaAlignment: center
coverMeta: out
draft: true
---

参考[An overview of next-generation graphics APIs](http://nextgenapis.realtimerendering.com/)，理解并学习下一代graphics apis（vulkan、mental、dx12）的设计思想；
<!--more-->

## 为什么需要下一代GPU

1. 减少CPU过载或CPU瓶颈；在图形学中有一个优化指标称之为draw call，在上一代图形api中，只能以单线程的形式触发draw函数；在GPU强大的情况下，会出现cmd queue中的指令被GPU执行完毕了，CPU仍未完成下一个cmd queue的cmd填充；下一代GPU的解决方案既是支持多线程填充cmd queue，可以支持多线程填充多个cmd queue，也即支持在多线程中调用图形api接口；这样cmd queue的填充效率就大大提高，从而使GPU避免空闲的状况；
2. 获取更稳定的，可预测的驱动性能；在上一代图形api中，submit a drawcall或map a buffer，可能会涉及到shader编译、insert fence、flushing cache等操作，从而导致两个同样的api函数可能发生不同的操作，从而产生不同的耗时；这种情况下，很难获得一致且稳定的耗时；
3. 显示的CPU/GPU同步机制，显示的内存管理机制；上一代的graphics apis会自动完成这一模块，好处是使用者不用花过多的心思在这上面；带来的问题也必然是，限制了使用者的发挥，使用者不能根据具体情况来优化同步问题与内存管理问题；

## 下一代GPU的设计概念

1. 