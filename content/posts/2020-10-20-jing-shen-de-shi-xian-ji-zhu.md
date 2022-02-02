---
title: '景深的实现技术'
date: 2020-10-20 16:47:41
categories:
- 总结
tags:
- 总结
- DOF
metaAlignment: center
coverMeta: out
draft: true
---
景深的实现技术有很多，针对不同的使用场景，可以使用不同的方法；
<!--more-->

## 基于光线追踪的景深效果（离线）

基于光线追踪的景深效果，直接使用薄透镜模型，在透镜上面进行多采样即可实现景深效果；

关于薄透镜理论的使用，可以参考这里[基于摄影参数渲染](https://zhuanlan.zhihu.com/p/23827065);

## 基于累积贴图的景深效果（实时）

大致思路为，将相机进行移动（可按照透镜多采样的方式移动），沿焦平面进行多个相机的渲染，然后将渲染结果进行累加，这样就能获取与光线追踪类似的效果；本质上类似于光线追踪的多采样方式，但需要花费大量的DC，一般只用来验证；

## 基于分层绘制的景深效果（实时）

本质上，是基于2D图层的方式来实现；将场景按深度进行分层绘制，然后将远离焦距的绘制rt记性模糊，然后按层进行混合，即可获取接近景深的效果；

使用要求时，不同的景物之间不能有交叉，即物体不能有太强的深度变化；因为针对单个物体是无法产生即聚焦又失焦的现象；

## 基于前向映射的Z-buffer的景深效果（实时）

此方法常用在后处理效果中，该方法存储颜色缓冲与深度缓冲作为最后的blit对象；然后使用深度缓冲计算COC（circle of confusion），即点投影在屏幕上形成的弥散圆；再然后利用弥散圆进行模糊与blend，这里模糊并不是通常意义上的模糊，模糊需要的圆盘采样与普通模糊一致，但是采样的判定需要根据采样点的COC是否能覆盖到当前点，来确定该采样点的弥散圆是否对当前点有贡献；GPU Gems中说blend只能混合到距离摄像机比自己远的那些相邻像素中，以避免模糊的像素影响它们前面的清晰像素。实际上，blend的是为了避免前面模糊造成聚焦物体边缘的消失；

## 基于反向映射的Z-buffer的景深效果（实时）

该方法与上一种技术类似，区别在于并不是通过blend来形成最后的图像，而是通过使用深度值距焦距的距离来进行blur，从而形成最终的效果；这里把当前点的弥散圆当做模糊范围来进行计算了，与实际的PBR有些偏差，但也能凑活使用；

## Reference

1. [Depth of Field](https://catlikecoding.com/unity/tutorials/advanced-rendering/depth-of-field/)
2. [Depth of Field: A Survey of Techniques](https://developer.nvidia.com/gpugems/gpugems/part-iv-image-processing/chapter-23-depth-field-survey-techniques)
3. [基于摄影参数渲染](https://zhuanlan.zhihu.com/p/23827065)
4. [渲染中的景深(Depth of Field/DOF)](https://zhuanlan.zhihu.com/p/146143501)
5. [A Life of a Bokeh - SIGGRAPH 2018](https://epicgames.ent.box.com/s/s86j70iamxvsuu6j35pilypficznec04)