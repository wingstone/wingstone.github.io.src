---
title: '皮肤渲染——Screen Space Separable Subsurface Scattering'
date: 2020-09-16 11:31:29
tags: [Rendering,图形学]
published: true
hideInList: false
feature: 
isTop: false
---
总体来说，渲染步骤为：

1. 首先对于不同的皮肤散射颜色，去计算相应的kernel（作为一维的颜色数组，存储该距离下次表面散射贡献），kernel是通过多层高斯曲线去拟合偶极子曲线得到的，因此需要多次高斯叠加计算；kernel的长度大小对应模糊的范围，也对应采样数的大小；
2. 正常渲染皮肤，但要使用MRT，将diffuse成分与specular分离，并使用stencil进行标记；
3. 在相机的BeforeImageEffectsOpaque时，进行皮肤部分的separable blur，并使用stencil test，保证只在皮肤部分进行blur；blur时要使用前面计算出来的kernel以及模型的曲率（借助深度的ddx、ddy计算），依据曲率来调整实际blur的范围；
4. 将MRT进行合并，即blur后的diffuse部分与specular部分进行结合；

## Reference
1. [Separable Subsurface Scattering](http://iryoku.com/separable-sss/)
2. [Advanced Techniques for Realistic Real-Time Skin Rendering](https://developer.nvidia.com/gpugems/gpugems3/part-iii-rendering/chapter-14-advanced-techniques-realistic-real-time-skin)