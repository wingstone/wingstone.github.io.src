---
title: 'volumetric fog'
author: wingstone
date: 2022-08-22
categories:
- contents
metaAlignment: center
coverMeta: out
---

体积雾简单实现尝试

<!--more-->

实现方式主要参考[Physically-based & Unified Volumetric Rendering](https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite)；


整个Volumetric Pipeline的构建中，分成了三种Volume的构建，分别为

1. material parameter volume
2. Inscattering extinction volume
3. Scattering transmittance volume

整个raymatching过程，只在view方向进行，light方向的scattering与absorption暂不考虑。

在实现上，最大的优化为：在raymatching步进段，可认为inscatter light为常量，但是认为transmittance为常量是不合理，会带来较大误差，直接导致能量不守恒。若以步进段的起始点为常量，会导致transmittance偏小，若以步进段的终点为常量，会导致transmittance偏大；

因此，当前步进段上，inscatter light的积分应使用解析解来计算。其他步骤保持不变。Inscattering的计算需要考虑可见性，使用shadowmap来计算。从而带来godray、sunshaft效果。

material parameter volume的使用，使得artist能够有更高的自由度来控制体渲染；

由于volume的分辨率不足，最终的结果会有artifact，使用view ray方向上的jitter，配合temporal filter来达到降噪与提升分辨率的效果；