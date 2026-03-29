---
title: '浅水方程&波动方程'
author: wingstone
date: 2026-03-05
categories:
- water interaction
tags: 
- fluid physics
draft: true
---

归纳总结在gpu上试下流体模拟，主要总结gpu gems1上的一篇文章，在2d平面上模拟流体实现；

<!--more-->

## 前置理论知识

浅水方程所采用的假设：

1. 浅水假设：水体的垂直尺度（水深h）远小于流动的水平尺度（如波长）h/L≪1。这允许我们将三维问题简化为二维（平面）问题。
2. 静水压力假设：垂直加速度远小于重力加速度，垂直方向上的压力分布近似服从静水压力分布。即水中某点的压力P仅取决于该点上方水柱的重量。
3. 无粘性（Inviscid）：忽略流体的粘性应力，即不考虑水流内部的摩擦。
4. 无底部摩擦：忽略水流与河床/海底之间的切向应力。这是用户明确指定的条件。
5. 流体不可压缩且均质：即密度为常数。
6. 假设地面平坦。

基于以上假设，即可对NS方程在高度上面进行积分，此时方程所对应的微元是一个竖条，上到水平面，下到水底，最终得到我们所看到的浅水方程：

连续性方程为：

$$
\frac{\partial H}{\partial t} + \frac{\partial \left( H u \right)}{\partial x} + \frac{\partial \left( H v \right)}{\partial y} = 0
$$

动量方程则为：

$$
\frac{\partial u}{\partial t} + u\frac{\partial u}{\partial x} + v\frac{\partial u}{\partial y} = -g\frac{\partial H }{\partial x}
$$
$$
\frac{\partial v}{\partial t} + u\frac{\partial v}{\partial x} + v\frac{\partial v}{\partial y} = -g\frac{\partial H }{\partial y}
$$

其中 $H = \eta + d$ ，d为水平面水深， $\eta$ 为水面据水平面的距离，一般取水平面为z为0的位置。

若忽略动量方程中的对流项，即可得到波动方程，即：

$$
\frac{\partial u}{\partial t} = -g\frac{\partial \eta }{\partial x}
$$
$$
\frac{\partial v}{\partial t} = -g\frac{\partial \eta }{\partial y}
$$

## 解算波动方程

将连续性方程对时间t求导，可得：

$$
\frac{\partial ^2 \eta }{\partial ^2 t} + d \frac {\partial}{\partial t} \left( \frac{\partial \left( \eta u \right)}{\partial x} + \frac{\partial \left( \eta v \right)}{\partial y} \right)= 0
$$

将波动方程带入上式可得：


$$
\frac{\partial ^2 \eta }{\partial ^2 t} = gd \left( \frac{\partial ^2 u}{\partial x} + \frac{\partial ^2 v }{\partial y} \right)
$$

使用有限差分法可得：

$$
\eta ^{n+1}_{i,j} = 2\eta ^n_{i,j} - \eta ^{n-1}_{i,j} +\frac{gd \Delta t^2}{\Delta x^2} \left( \eta ^n_{i-1,j} + \eta ^n_{i+1,j} + \eta ^n_{i,j-1} + \eta ^n_{i,j+1} - 4\eta ^n_{i,j} \right)
$$

如此即可得到高度随位置以及时间的变化；
参考王华民老师的课程，上式可以调整为：

$$
\eta ^{n+1}_{i,j} = \eta ^n_{i,j} + \beta\left( \eta ^n_{i,j}- \eta ^{n-1}_{i,j} \right) +\alpha \left( \eta ^n_{i-1,j} + \eta ^n_{i+1,j} + \eta ^n_{i,j-1} + \eta ^n_{i,j+1} - 4\eta ^n_{i,j} \right)
$$

其中 $\alpha$ 与 $\beta$ 为两个常数，额外添加的 $\beta$ 参数可以调整时间对动量的影响程度，起到粘性的作用；而 $\alpha$ 参数则为与 $\Delta t$ $\Delta x$有关的常数； 还可以添加一个Damping参数来影响波的衰减，如下所示：

$$
\eta ^{n+1}_{i,j} = Damping * \eta ^{n+1}_{i,j}
$$

Damping参数为0-1的值，如此便能暴力的保证最终的水平面可以归0，同时能加快波的衰减；

## 解算浅水方程



## Reference

1. [GAMES103-基于物理的计算机动画入门](https://www.bilibili.com/video/BV12Q4y1S73g/?p=10&spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=4d028a6c1255cabb85fd480d6d5e54d8)
2. [Shallow water equations wiki](https://en.wikipedia.org/wiki/Shallow_water_equations#cite_note-3)