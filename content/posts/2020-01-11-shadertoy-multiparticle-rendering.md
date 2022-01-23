---
title: 'ShaderToy MultiParticle Rendering'
date: 2020-01-11 14:31:41
categories:
- Shadertoy
tags: 
- Shadertoy
- Particle
---

在ShaderToy开发过程中，使用粒子可以极高的提升粒子效果，绘制粒子可以分为**粒子着色**以及**粒子范围的确定**两部分，这篇文章主要讨论**粒子范围的确定**。
<!--more-->

## 粒子范围的确定

假如有40个粒子，最简单方法就是循环遍历四十个粒子的位置及半径，每遍历一个粒子就对一个粒子着色，这样就能绘制出一个简单的粒子系统，iq所写的[Bubble](https://www.shadertoy.com/view/4dl3zn)就是遍历40个粒子，来模拟大量泡泡的上升效果;
其中位置及范围确定的伪代码为：

```c++
for( int i = 0; i++; i < 40 )
{
    bool inCircle = distance(curPos, particlePos[i]) < particleRedius[i];
}
```

## 循环的优化

ShaderToy中开发着色器完全是在fragment shader中执行的，所以我们写的程序是在每个像素中执行一次，如何让程序在一个像素中执行时间变短是我们优化的目标；在此例子中，只要我们降低了计算的循环次数就能优化效率；

在[worley noise的实现](https://zhuanlan.zhihu.com/p/94632440)中，有一个很好的思想就是限制控制点在栅格内，这样计算最近距离时，只需要计算最近的9个栅格即可；

将方法映射到粒子的计算，我们可以将粒子中心限制到一个栅格，半径不超过一个栅格的边长，这样我们只需要计算最近的9个栅格就能完成粒子的绘制；

这样会限制粒子的覆盖范围，因此我们可以将栅格的边长划分的大一些，或者遍历25个栅格，而扩大粒子的覆盖范围；

但是这种方法只是扩大了粒子的覆盖范围，对于粒子的运动范围，粒子的中心点必须在中心的栅格内运动才不会出现割裂的效果，因此唯一的运动方法就是移动整体的UV；

移动整体的UV会导致粒子的运动过于单一，可改进的方法为使用多层粒子叠加，这样不同层的移动效果不一样，倒是会增加一点真实感~

shader范例：
<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/WtK3D1?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>
