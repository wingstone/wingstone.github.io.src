---
title: '浅水方程&波动方程'
author: wingstone
date: 2026-03-05
categories:
- water interaction
tags: 
- fluid physics
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
\frac{\partial H}{\partial t} + \frac{\partial \left( Hu \right)}{\partial x} + \frac{\partial \left( Hv \right)}{\partial y} = 0
$$

动量方程则为：

$$
\frac{\partial u}{\partial t} + u\frac{\partial u}{\partial x} + v\frac{\partial u}{\partial y} = -g\frac{\partial H}{\partial x}
$$
$$
\frac{\partial v}{\partial t} + u\frac{\partial v}{\partial x} + v\frac{\partial v}{\partial y} = -g\frac{\partial H}{\partial y}
$$

若忽略动量方程中的对流项，即可得到波动方程，即：

$$
\frac{\partial u}{\partial t} = -g\frac{\partial H}{\partial x}
$$
$$
\frac{\partial v}{\partial t} = -g\frac{\partial H}{\partial y}
$$

## 解算波动方程

将连续性方程对时间t求导，可得：

$$
\frac{\partial ^2 H}{\partial ^2 t} + h \frac {\partial}{\partial t} \left( \frac{\partial \left( Hu \right)}{\partial x} + \frac{\partial \left( Hv \right)}{\partial y} \right)= 0
$$

将波动方程带入上式可得：


$$
\frac{\partial ^2 H}{\partial ^2 t} = gh \left( \frac{\partial ^2 u}{\partial x} + \frac{\partial ^2 v }{\partial y} \right)
$$

使用有限差分法可得：

$$
H^{n+1}_{i,j} = 2H^n_{i,j} - H^{n-1}_{i,j} +\frac{gh \  t}{\partial y}
$$



## Reference

1. [GAMES103-基于物理的计算机动画入门](https://www.bilibili.com/video/BV12Q4y1S73g/?p=10&spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=4d028a6c1255cabb85fd480d6d5e54d8)
2. [Shallow water equations wiki](https://en.wikipedia.org/wiki/Shallow_water_equations#cite_note-3)