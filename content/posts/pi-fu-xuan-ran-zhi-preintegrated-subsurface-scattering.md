---
title: '皮肤渲染——Preintegrated Subsurface Scattering'
date: 2020-09-28 12:18:24
tags: [图形学,Rendering]
published: true
hideInList: false
feature: 
isTop: false
---
预积分皮肤散射主要解决三种皮肤散射情况：
1. 表面弯曲引起的散射（Surface Curvature）；
2. 表面小凸起引起的散射（Small Surface Bumps）；
3. 投影边缘引起的散射（Shadows）；
4. 皮肤背面的透射问题（Translucency）（自己添加的）；
<!--more-->

## Surface Curvature

单纯的wrap并不符合物理，需要通过diffusion profile积分才能获取正确的wrap；
预积分贴图是在球形假设下进行计算的，因此该方法最大的缺陷是模拟表面拓扑复杂的结构时，有很大不合理之处；
幸运的是，当今模型表面大都是平滑的，而不平滑的小的部分都用法线贴图进行假设计算；
积分方程为：

$$
D(\theta, r) = \frac{\int_{-\pi/2}^{\pi/2} {cos(\theta+x)*R(2rsin(x/2))} \,{\rm d}x}{\int_{-\pi/2}^{\pi/2} {R(2rsin(x/2))} \,{\rm d}x} 
$$

式中R(d)表示Diffusion profile，表示相应距离下的辐射度，d表示距离。该式表示在曲率半径为r的半球下，对应$\theta$角度下，其它所有角度在该角度下的散射强度；对应模型如下图所示：

![Diffusion profile，Integrated model](https://wingstone.github.io/post-images/1601282044664.jpg)

**图中的$\theta$应该全部用x来表示！**从图中的右侧的模型能够看出，N为我们要求的散射角度，L与N之间的角度为$\theta$，L在$N+x$处的光照强度为$cos(\theta+x)$；N距N+x的距离为2rsin(x/2)，即弦长；N+x在N处的散射可由R(d)计算得出；

图中的Diffusion profile一般使用高斯核叠加进行表示，对于rgb各成分的高斯核表示为：
![A Sum-of-Gaussians Fit for Skin](https://wingstone.github.io/post-images/1601282423542.jpg)

最终得到一个Diffusion profile在不同$\theta$角度和曲率半径下的积分分布；将$\theta$角度和曲率半径转换为NdoL和1/r，即可得到常用的预积分贴图；如下图所示：
![曲率，预积分贴图](https://wingstone.github.io/post-images/1601283515546.jpg)

其中曲率的求法可在shader中借助偏导函数计算，即：
```C++
float3 dn = fwidth(N);
float3 dp = fwidth(P);

float c = length(dn)/length(dp);
```

## Small Surface Bumps

对于小的凸起以及皱纹，不能使用预积分来进行计算，但是由于可以多采样，因此可以使用预滤波的方式处理法线贴图；

对于specular使用正常的normalmap，对于diffuse的rgb分别使用针对不同Diffusion profile处理后的normalmap，因此需要4张normalmap；

为了效率考虑，可以使用一种高精度无滤波的normalmap，一张低精度预滤波的normalmap分别针对rgb插值来近似相应的预滤波处理；需要2张normalmap；

可以将低精度无滤波的normalmap省略，直接使用模型法线来代替，这样就可以指使用一张高精度无滤波的normalmap进行处理了；

最终的结果为：进行滤波处理的通道，其法线凹凸变弱，反应在视觉上，就是凹凸处该通道有相应的溢出（根本原因是此通道趋于恒定，而其他通道凹凸变换大）；

## Shadows Scattering

皮肤出的散射特性在投影边缘会体现出来，具体模拟此现象是很困难的一件事情，但是我们可以通过一些trick来实现；

首先，阴影强度为0或1，表示被遮挡，或不被遮挡；位于两者之间的值，及表示半影区域，及发生散射的区域（前提是使用软阴影）；因此我们可以针对这一点，利用Diffusion profile进行积分，来得到阴影值与散射强度的关系；

积分的过程是针对位置进行积分，需要将阴影值映射到位置上后才能积分，针对不同的软阴影方法，映射函数是不同的；我们用P()表示阴影值与到blur kernel距离的函数，则此映射为$p^-{1}()$；具体的积分方程为：
$$
P_S(s,w)  = \frac{\int_{-\infty}^{\infty} {P^'(P^{-1}(s)+x)R(x/w)} \,{\rm d}x}{\int_{-\infty}^{\infty} {R(x/w)} \,{\rm d}x} 
$$
针对box blur软阴影（PCF）方法，其过程如下：
![投影预积分](https://wingstone.github.io/post-images/1601347525242.jpg)

需要注意的是，半影区域的一部分（宽度到没提及）作为正常的软阴影计算插值，剩下的一部分作为这些软阴影散射产生的影响；P'()即表示新的半影函数分布；

最终得到一张投影的预积分图；其散射强度为阴影值s、以及半影宽度w（世界空间）的函数；

## Translucency

关于背面透射问题，可以借助light空间的depth map来大约计算光线在物体所穿过的距离；使用depth map对于非凸物体会产生一些不正确现象，但是还能够接受；

使用depth map需要重新渲染一张深度图，直接使用shadowmap其实也可以，但对于某些管线并不能很好的结合在一起，因此，一般使用一张表示物体的厚度贴图来作为光线穿过距离；

使用指数函数来处理距离，典型的使用`exp(-depth * sigma_t)`即可，这里sigma_t表示物体的透射系数；

额外需要处理的问题包括：
1. 光照的正面会得到穿过距离为0的问题，需要使用ndl进行处理，不能只是用ndl的sign值，否则会产生锯齿，不能产生平滑结果；
2. 同理只有视角背向光源时，才能产生透射现象，需要使用vdl进行处理；

最终的渲染结果为：
![预积分渲染结果](https://wingstone.github.io/post-images/1601388479937.jpg)

## 问题及展望

1. 曲率在PS中计算的话，在三角形边缘会出现异常；
2. 三张normal map贴图问题；
3. 对几何拓扑做出了大量的假设；

## Reference
1. [Penner pre-integrated skin rendering (siggraph 2011 advances in real-time rendering course)](https://www.slideshare.net/leegoonz/penner-preintegrated-skin-rendering-siggraph-2011-advances-in-realtime-rendering-course)：PPT
2. GPU Pro 2, Part 2. Rendering, Chapter 1. Pre-Intergrated Skin Shading：Book
3. [Simon's Tech Blog](http://simonstechblog.blogspot.com/2015/02/pre-integrated-skin-shading.html)：使用数值插值代替预积分贴图
4. [GPU Gems 1, Real-Time Approximations to Subsurface Scattering](https://developer.nvidia.com/gpugems/gpugems/part-iii-materials/chapter-16-real-time-approximations-subsurface-scattering)