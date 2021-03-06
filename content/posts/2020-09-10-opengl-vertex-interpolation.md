---
title: 'Opengl Vertex Interpolation'
date: 2020-09-10 14:39:18
categories:
- 图形学
tags:
- Unity
- GPU
- 心得
metaAlignment: center
coverMeta: out
---

OpenGL中关于插值问题
<!--more-->

## OpenGL中关于插值问题

在GPU的光栅化阶段，会针对每个片段进行插值计算；需要注意的是，这个时候进行的插值不能是简单的screen space线性插值，而应该是透视空间矫正的view space线性插值；

OpenGL中默认的是透视插值，也就是在view space空间中的线性插值，可以给变量加no perspective in/out使其插值时不做透视矫正，也就不线性了。如果要自定义插值方式，只能自己把相关参数都传到片元shader里计算了。

> 注意：透视矫正插值仍是线性的，只不过是screen space下等距的线性插值，转换为view space下的非等距的线性插值，也是view space空间上的线性插值；

## 透视插值的公式推导

参考《3D游戏与计算机图形学中的数学方法》4.4节

简单概括就是：

对于深度插值：由相似三角形变换可以得到，Z的倒数在光栅化时，恰好是按照线性进行插值的；

对于顶点属性插值：也能推导出：
$$
b_3 = \frac{\frac{b_1}{z_1}(1-t)+\frac{b_2}{z_2}t}{\frac{1}{z_1}(1-t)+\frac{1}{z_2}t}
$$
也就是说，顶点属性的插值为$b/z$的线性插值再除以$1/z$的线性插值；

## 为什么像素插值光照要优于顶点插值光照

是因为顶点插值光照是线性插值（当然指透视矫正后）的，但是对于向量只是进行线性插值是不够的，还应该有个归一化阶段；

像素插值光照可以在PS中进行向量的归一化（特别是法向量），如此就纠正了向量插值的错误问题，然后可进行正确的光照；

其次光照计算过程并不是一个线性的过程；对计算结果进行线性插值，当然就不可能得到正确的光照效果；
