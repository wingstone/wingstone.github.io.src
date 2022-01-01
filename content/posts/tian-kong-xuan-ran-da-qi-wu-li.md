---
title: '天空渲染——大气物理'
date: 2020-10-10 15:38:30
tags: []
published: true
hideInList: false
feature: 
isTop: false
---

## 大气物理现象

假设1：空气密度随着高度成指数进行衰减；即：
$$
density(h) = density(0)e^{-\frac{h}{H}}
$$
其中density(0)为海平面密度，h为当前高度，H为scale height，随温度变化；不是所有的人都采用这种模型，另一种密度假设为分层模型，每一层采用不同的大气密度；

大气分子散射主要可分为**空气分子散射（Rayleigh scattering，小于光线波长）**与**气溶胶散射（Mie scattering，大于光线波长）**；空气分子散射主要会产生天空的蓝色成分与橘黄色成分（日出、日落时分），气溶胶散射主要会导致灰白色条带（特别是污染都市上空）；

在进行光追计算时，还需要进行地球半径的假设，以及大气层半径的假设；我们假设地球半径为6360 km，大气半径为6420 km，此外还需要假设太阳光为平行光，因为太阳距地球足够远；

计算的过程中需要注意量纲的统一，一般都去Km；

熟悉体渲染的人应该知道，物体的体渲染熟悉可以用散射系数（针对外散射）、吸收系数、相位函数（针对内散射）进行描述，我们这里忽略大气吸收的过程，即忽略吸收系数的计算；

## Rayleigh scattering

Rayleigh scattering具有很强的波长依赖性，散射蓝光比绿光和红光具有更高的精确性；

Rayleigh首先提出了计算此现象的公式，他给出了散射系数的公式：
$$
{\beta}_R^s(h, \lambda) = \frac{8\pi^3(n^2-1)^2}{3N\lambda^4}e^{-\frac{h}{H_R}}
$$

这里$\lambda$表示波长，N表示大气分子密度，n为大气折射率，$H_R$即为前面的scale height，此处我们取8Km；

参考文献中N与n并没有给出相应的参数，但是给出了相应的散射系数$\beta^s_R=(5.8, 13.5, 33.1)10^-6m^-1$ for $\lambda=(680, 550, 440)nm$；

渲染使用的extinction coefficient为：
$$
\beta_R^e = \beta_R^s
$$

相位函数的公式为：
$$
P_R(\mu)=\frac{3}{16\pi}(1+\mu^2)
$$
其中$\mu$表示光线与视角的夹角；

## Mie Scattering

与Rayleigh Scattering类似，其散射公式为：
$$
\beta_M^s(h,\lambda)=\beta_M^s(0,\lambda) e^{-\frac{h}{H_M}}
$$
这里的$H_M$通常取1.2Km；同样类似于Rayleigh Scattering，气溶胶密度也随高度进行指数衰减，我们直接取散射系数为$\beta^s_M=210x10^-5m^-1$；

渲染使用的extinction coefficient为：
$$
\beta_R^e = 1.1\beta_R^s
$$

相位函数为：
$$
P_M(\mu)=\frac{3}{8\pi}\frac{(1-g^2)(1+\mu^2)}{(2+g^2)(1+g^2-2g\mu)^{\frac{3}{2}}}
$$
其中$g$项用来控制介质的各向异性，一般取0.76；


## Reference

[寒霜引擎天空渲染实现](https://www.ea.com/frostbite/news/physically-based-sky-atmosphere-and-cloud-rendering)

[Simulating the Colors of the Sky](https://www.scratchapixel.com/lessons/procedural-generation-virtual-worlds/simulating-sky)

[Precomputed Atmospheric Scattering](http://www-ljk.imag.fr/Publications/Basilic/com.lmc.publi.PUBLI_Article@11e7cdda2f7_f64b69/article.pdf)