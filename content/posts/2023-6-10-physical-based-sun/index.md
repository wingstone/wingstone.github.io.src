---
title: 'physical based sun'
author: wingstone
date: 2023-06-10
categories:
- unreal
tags: 
- pbr
---

unreal中基于物理的sun light使用，主要参考材质蓝图中`SkyAtmosphereLightDiskLuminance`节点以及`SkyAtmosphereLightIlluminance`节点的实现，代码在`MaterialTemplate.ush`中；

<!--more-->

## 前置理论知识

关于体渲染相关的理论知识可参考[文章](/content/posts/2022-08-12-physically_based_atmosphere_scattering/)；

## sun disk实现

知道了理论知识后，可知sundisk的luminance由太空中sundisk的luminance，乘以大气产生的transmittance得到，即：

```c++
LightDiskLuminance = TransmittanceToLight * AtmosphereLightDiscLuminance;
```

如何得到太空中sundisk的luminance，如果直接使用directional light中经常设置的6lux，3.14lux，物理上就错了，lux是**illuminance**的物理量，并不是我们需要的**luminance**物理量，属于基于物理的大乌龙；

在Unreal中实现代码如下：

```c++
// SkyAtmosphereRendering.cpp
static FLinearColor GetLightDiskLuminance(FLightSceneInfo& Light)
{
	
	const float SunSolidAngle = 2.0f * PI * (1.0f - FMath::Cos(Light.Proxy->GetSunLightHalfApexAngleRadian()));			// Solid angle from aperture https://en.wikipedia.org/wiki/Solid_angle 
	return  Light.Proxy->GetAtmosphereSunDiskColorScale() * Light.Proxy->GetOuterSpaceIlluminance() / SunSolidAngle;	// approximation
}

void PrepareSunLightProxy(const FSkyAtmosphereRenderSceneInfo& SkyAtmosphere, uint32 AtmosphereLightIndex, FLightSceneInfo& AtmosphereLight)
{
	// See explanation in "Physically Based Sky, Atmosphere	and Cloud Rendering in Frostbite" page 26
	const FSkyAtmosphereSceneProxy& SkyAtmosphereProxy = SkyAtmosphere.GetSkyAtmosphereSceneProxy();
	const FVector AtmosphereLightDirection = SkyAtmosphereProxy.GetAtmosphereLightDirection(AtmosphereLightIndex, -AtmosphereLight.Proxy->GetDirection());
	const FLinearColor TransmittanceTowardSun = SkyAtmosphereProxy.GetAtmosphereSetup().GetTransmittanceAtGroundLevel(AtmosphereLightDirection);

	const FLinearColor SunDiskOuterSpaceLuminance = GetLightDiskLuminance(AtmosphereLight);

	AtmosphereLight.Proxy->SetAtmosphereRelatedProperties(TransmittanceTowardSun, SunDiskOuterSpaceLuminance);
}

```

其中Light.Proxy->GetOuterSpaceIlluminance()，即在编辑器里面设置的light color与intensity的乘积；illuminance除以太阳对应的solidangle，即可得到我们需要的luminance；

> 注意：我们所设置的light color与intensity针对的是大气层表面太空的illuminance，不是地球表面的iluminance；

## sun light实现

由于太阳的光照需要考虑整个sun disk的影响，所以一般使用illuminance来控制太阳的亮度；同样，考虑大气的影响，地球表面的illuminance为：

```c++
LightIlluminance = AtmosphereLightIlluminanceOuterSpace * TransmittanceToLight;
```

其中，AtmosphereLightIlluminanceOuterSpace即为编辑器里面设置的light color与intensity的乘积；最终得到的LightIlluminance，即为最终渲染方程中的L；

值得注意的是，渲染方程为：

$$
L_o(l) = \int_H {L_i(l)f(l,v)cos\theta_l} {\rm d}\theta
$$

其中Li与Lo皆为luminance；而我们得到的是illuminance，是不能直接带入到方程的；

但是sun disk对应的角度很小，可以认为f与cos为常数，那么方程可改写为：

$$
L_o(l) = f(l,v)cos\theta_l \int_H {L_i(l)} {\rm d}\theta
$$

其中

$$
\int_H {L_i(l)} {\rm d}\theta
$$

为luminance沿角度的积分，即为我们所得到的LightIlluminance；也即我们设置时所改变的物理量；同时也简化了渲染方程的计算；

## Reference

1. [Byte alignment and ordering](https://eventhelix.com/embedded/byte-alignment-and-ordering/#:~:text=General%20Byte%20Alignment%20Rules%201%20Single%20byte%20numbers,%20total%20structure%20is%204%20bytes.%20More%20items)
2. [Purpose of memory alignment](https://stackoverflow.com/questions/381244/purpose-of-memory-alignment)
3. [Data alignment: Straighten up and fly right](https://developer.ibm.com/articles/pa-dalign/)
4. [The Lost Art of Structure Packing](http://www.catb.org/esr/structure-packing/)