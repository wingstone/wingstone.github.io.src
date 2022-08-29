---
title: 'metric in pbr'
author: wingstone
date: 2022-08-22
categories:
- pbr
metaAlignment: center
coverMeta: out
---

pbr渲染中涉及到的单位问题，参考GAMES101课程；

<!--more-->

## Radiometry（辐射度学）
辐射度量学是基于物理渲染的理论基础，它将渲染使用物理下的单位及公式进行表达，使得PBR能够站住脚跟；

 - Energy（能量）：单位J焦耳；
 - Flux（通量）：定义为单位时间内通过的能量，单位W瓦特（lm流明，光度学，两者等价）；
 - Intensity（强度）：定义为单位立体角下单位时间内通过的能量，单位cd坎德拉；
 - Irradiance（辐照度）：定义为单位投影面积下单位时间内通过的能量，单位lux拉克丝；由于是投影面积，因而伴随着Lambert定律；对于点光源，存着辐照度随着半径平方衰减定律；
 - Radiance（辐射率）：定义为单位投影面积下单位体积角下单位时间内通过的能量，单位nit尼特；用来描述光线的单位，渲染整个过程都是在进行Radiance的计算；

Radiance是与距离无关的，不考虑fog或其他体积介质的话；
Radiance的物理实质应该是有相应的SPD来表示，但是常用RGB来进行表示；使用RGB来表示的原因可以参考文章[Color and Perception](https://blog.csdn.net/qq_35817700/article/details/117934398)

### Irradiance vs Radiance

Irradiance指单位面积下单位时间内接收的的辐射能量；
Radiance指单位面积下某个单位角度下单位时间内接收的的辐射能量；

### Bidirectional Reflectance Distribution Function(BRDF)

BRDF代表的为反射点某个出射角度下Radiance与某个入射角度下Irradiance的比值（即BRDF函数是有物理单位的）；由于入射Irradiance表示单位面积下的辐射通量，为了递归光线追踪，常需要将其表示为入射Radiance的微分形式；
BRDF的单位应为radiance/irridiance，即1/sr，表示方向的度量；

### Camera
一般相机模型下渲染积分得到的结果为**Radiance**，我们得到的是单位面积下某个方向上的单位立体角下的辐射通量；
对于真实的相机，得到结果为**Irradiance**，因为真实的传感器需要累积单位面积下各个方向上的辐射通量（比如整个光圈所对应的在该传感器点的体积角）；可能真实的相机也会进行角度的平均，来得到Radiance；

## Photometry（光度学）

辐射度学使用的是完全基于物理的度量，但是实际上人眼对不同波长的光强敏感性是不一样的（眼睛中杆细胞的存在，同样可参考后面的文章[Color and Perception](https://blog.csdn.net/qq_35817700/article/details/117934398)）；
光度学与辐射度学类似，除了它对于所有的东西都使用了人眼的强度敏感函数进行了加权；
因此，辐射度学所计算出来的结果，需要考虑眼睛对波长的敏感性，转之后的结果就属于光度学下的度量；
Photometry与Radiometry唯一的不同在于转换曲线以及使用单位；单位差异如下：
Radiometry Units | Photometry Units
---------|--------
radiant flux: watt(W) | Luminous flux: lumen(lm)
irradiance: W/m^2 | illuminance: lux(lx)
radiant intensity: W/sr | luminous intensity: candela (cd)
radiance: W/(m^2sr) | luminance: cd/m^2 = nit