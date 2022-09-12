---
title: 'Computer shader介绍'
author: wingstone
date: 2022-08-29
categories:
- contents
metaAlignment: center
coverMeta: out
---

Computer shader介绍

<!--more-->

Compute shader的优势，
- 可以不执行完整的graphic管线，只运行compute units。
- 可以利用更快速的shared memory进行一些运算。
- 可以利用async机制，充分利用gpu效率。

# 关于async机制

Gpu在做线程组切换时，涉及到状态的更新，会导致Compute unit空闲，从而导致gpu运算能力的浪费，额外增加latency。
async cs提供异步执行Compute shader的能力，可以在线程组切换的时候插入其他的线程的能力。从而提高gpu的利用率。
一般来说，在任务比较重的graphic queue中插入轻量的compute queue与copy queue能更好地减小gpu latency。
相关文章可参考[Async shader](https://developer.amd.com/wordpress/media/2012/10/Asynchronous-Shaders-White-Paper-FINAL.pdf)

另外shadowmap pass与predepth pass是两个很好的执行async compute shader的时机，因为这两隔pass都是属于compute unit消耗很少的pass，大部分的运行瓶颈在rop操作上，因此使用async compute shader可以充分利用compute unit；