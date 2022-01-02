---
title: Real Shading in Unreal Engine 4
author: wingstone
date: 2022-01-01
categories:
- SIGGRAPH
tags:
- unreal
- pbr
- realtime
metaAlignment: center
coverMeta: out
---

本文主要讲述PBR在Unreal中的实现思路，主要涉及Material Model、Shading Model、Lighting Model的背后原理与经验总结；在文章最后，添加了我个人的理解与实现过程中的相关参考；
<!--more-->

## 介绍

想要切换到pbr工作流的原因：可以渲染出更加真实的照片，同时能极大地改善美术工作人员的工作流与工作质量。

受迪士尼的以前[Physically-Based Shading at Disney](http://blog.selfshadow.com/publications/s2012-shading-course/)的启发 ，unreal制定了自己的pbr所要达成的目标：

1. Real-time performance，足够高效，满足实时渲染的帧率要求，（移动端还需考虑温度，耗电量要求）
2. Reduced complexity，参数尽可能少，过多的参数会产生更多的试错成本与失误。由于要受到ibl与解析光源的光照，因此，参数参数必须具有跨光源下的统一性。
3. Intuitive interface，简单易理解的接口，避免折射率这种物理数值。
4. Perceptual Linearly，感知上线性，意味着参数blend后的结果要尽可能接近结果的blend；（个人感觉不太可能，pbr本身的计算并不是线性可插值的）
5. Easy to Master，易于精通，不需要理解太多技术即可完成物理可信的效果；
6. Robust，意味着，不易产生不符合物理的效果，切多个参数的混合仍能获得物理可信效果；
7. Expressive，富有表现力的，由于使用延迟管线，因此基础的光照模型需要表达真实世界约99%的材质；
8. Flexible，足够灵活，项目并不都是真实感渲染，需要灵活到能够承载非真实感渲染；

## Shading Model

### Diffuse BRDF

diffuse使用最简单的Lambert光照模型；

$$
f(l,v) = \frac{C_{diff}}{\pi}
$$

### Microfacet Specular BRDF

通用的Cook-torrance微表面模型为：

$$
f(l,v) = \frac{D(h)F(v,h)G(l,v,h)}{4(n \cdot l)(n \cdot v)}
$$

#### Specular D

D项起着非常重要的作用，直接影响着高光的trail分布，且采用GGX并不会比blinn-phong多出很多消耗，因此D项为：

$$
D(h) = \frac{\alpha^2}{\pi((n \cdot h)^2(\alpha^2+1)+1)^2}
$$

这里采用Disney所使用的的重参数化做法，即$\alpha=Roughness^2$；这里的Roughness在Unity的实现中对应于其中的ProcedualRoughness；

#### Specular G

G项对比下来，最终选用Schlick model，并且使用$k=\frac{\alpha}{2}$，如此来更好的逼近GGX中的Smith model；参考Disney的做法将Roughness重映射为$\frac{Roughness+1}{2}$来减少“hotness”，只适用于解析光源，对于ibl计算，由于是离线的，可直接使用smith model；

$$
k=\frac{(Roughness+1)^2}{8}
$$
$$
G_1(v)=\frac{n\cdot v}{(n\cdot v)(1-k)+k}
$$
$$
G(l,v,h)=G_1(l)G_1(v)
$$

#### Specular F

菲涅尔项同样采用Schlick 拟合项，并做了部分修改，使用了Spherical Gaussian approximation 来代替Power操作，差异是极其微小，但带来了性能的提升；

$$
F(v,h)=F_0+(1-F_0)2^{(-5.55473(v\cdot h)-6.98316)(v\cdot h)}
$$

## Image-Based Lighting

我们使用前面的Shading Model来处理IBL，正常情况下要使用重要性采样来进行；积分公式为：

$$
\int_H {L_i(L)f(l,v)cos\theta_l} \,{\rm d}l= \frac{1}{N}\sum_{k=1}^N \frac{L_i(l_k)f(l,v)cos\theta_l}{p(l_k, v)} \quad
$$

代码为：

```c++
float3 ImportanceSampleGGX( float2 Xi, float Roughness, float3 N )
{
    float a = Roughness * Roughness;
    float Phi = 2 * PI * Xi.x;
    float CosTheta = sqrt( (1 - Xi.y) / ( 1 + (a*a - 1) * Xi.y ) );
    float SinTheta = sqrt( 1 - CosTheta * CosTheta );
    float3 H;
    H.x = SinTheta * cos( Phi );
    H.y = SinTheta * sin( Phi );
    H.z = CosTheta;
    float3 UpVector = abs(N.z) < 0.999 ? float3(0,0,1) : float3(1,0,0);
    float3 TangentX = normalize( cross( UpVector, N ) );
    float3 TangentY = cross( N, TangentX );
    // Tangent to world space
    return TangentX * H.x + TangentY * H.y + N * H.z;
}

float3 SpecularIBL( float3 SpecularColor , float Roughness, float3 N, float3 V )
{
    float3 SpecularLighting = 0;
    const uint NumSamples = 1024;
    for( uint i = 0; i < NumSamples; i++ )
    {
        float2 Xi = Hammersley( i, NumSamples );
        float3 H = ImportanceSampleGGX( Xi, Roughness, N );
        float3 L = 2 * dot( V, H ) * H - V;
        float NoV = saturate( dot( N, V ) );
        float NoL = saturate( dot( N, L ) );
        float NoH = saturate( dot( N, H ) );
        float VoH = saturate( dot( V, H ) );
        if( NoL > 0 )
        {
            float3 SampleColor = EnvMap.SampleLevel( EnvMapSampler , L, 0 ).rgb;
            float G = G_Smith( Roughness, NoV, NoL );
            float Fc = pow( 1 - VoH, 5 );
            float3 F = (1 - Fc) * SpecularColor + Fc;
            // Incident light = SampleColor * NoL
            // Microfacet specular = D*G*F / (4*NoL*NoV)
            // pdf = D * NoH / (4 * VoH)
            SpecularLighting += SampleColor * F * G * VoH / (NoH * NoV);
        }
    }
    return SpecularLighting / NumSamples;
}

```

就算使用重要性采样，使用mipmap来加速收敛，仍至少需要16次采样才能满足效果，再考虑到反射采样之间的blend（各种反射探针），实际上采样一次才能满足性能要求；

### Split Sum Approximation

将积分进行分离即可达到预结算的效果，同时满足性能要求；

$$
\frac{1}{N}\sum_{k=1}^N \frac{L_i(l_k)f(l,v)cos\theta_l}{p(l_k, v)} \quad = \left(\frac{1}{N}\sum_{k=1}^N L_i(l_k) \quad \right)\left( \frac{1}{N}\sum_{k=1}^N \frac{f(l,v)cos\theta_l}{p(l_k, v)} \quad \right)
$$

### Pre-Filtered Environment Map

对于light部分，我们采用GGX来进行filter，将不同粗糙度下的filter结果存放到mipmap level中；同时假设$n=v=l$，引入的误差，在filter过程中使用$cos\theta_{l_k}$来加权得到更好的效果；

```c++
float3 PrefilterEnvMap( float Roughness, float3 R )
{
    float3 N = R;
    float3 V = R;
    float3 PrefilteredColor = 0;
    const uint NumSamples = 1024;
    for( uint i = 0; i < NumSamples; i++ )
    {
        float2 Xi = Hammersley( i, NumSamples );
        float3 H = ImportanceSampleGGX( Xi, Roughness, N );
        float3 L = 2 * dot( V, H ) * H - V;
        float NoL = saturate( dot( N, L ) );
        if( NoL > 0 )
        {
            PrefilteredColor += EnvMap.SampleLevel( EnvMapSampler , L, 0 ).rgb * NoL;
            TotalWeight += NoL;
        }
    }
    return PrefilteredColor / TotalWeight;
}
```

### Environment BRDF

第二项积分，可以认为是均匀白光下，对Specular brdf的积分，即$L_i(l_k)=1$；对于菲涅尔项，使用Schlick的形式：$F(v,h)=F_0+(1-F_0)(1-v\cdot h)^5$，我们可以发现，积分中$F_0$可以提取出来，即

$$
\int_H {f(l,v)cos\theta_l} \,{\rm d}l= F_0\int_H {\frac{f(l,v)}{F(v,h)}(1-(1-v\cdot h)^5)cos\theta_l} \,{\rm d}l + \int_H {\frac{f(l,v)}{F(v,h)}(1-v\cdot h)^5cos\theta_l} \,{\rm d}l
$$

最后的积分公式只需要Roughness与$cos\theta_v$作为输入，$F_0$的scale与bias作为输出。由于输入为0到1的范围，可以很容易的使用2dlut来存储积分结果。

Unreal使用R16G16float的存储格式来存储，因为测试发现精度起着非常重要的作用。

```c++
// 积分代码
float2 IntegrateBRDF( float Roughness, float NoV )
{
    float3 V;
    V.x = sqrt( 1.0f - NoV * NoV ); // sin
    V.y = 0;
    V.z = NoV; // cos
    float A = 0;
    float B = 0;
    const uint NumSamples = 1024;
    for( uint i = 0; i < NumSamples; i++ )
    {
        float2 Xi = Hammersley( i, NumSamples );
        float3 H = ImportanceSampleGGX( Xi, Roughness, N );
        float3 L = 2 * dot( V, H ) * H - V;
        float NoL = saturate( L.z );
        float NoH = saturate( H.z );
        float VoH = saturate( dot( V, H ) );
        if( NoL > 0 )
        {
            float G = G_Smith( Roughness, NoV, NoL );
            float G_Vis = G * VoH / (NoH * NoV);
            float Fc = pow( 1 - VoH, 5 );
            A += (1 - Fc) * G_Vis;
            B += Fc * G_Vis;
        }
    }
    return float2( A, B ) / NumSamples;
}

// 实时计算代码
float3 ApproximateSpecularIBL( float3 SpecularColor, float Roughness, float3 N, float3 V )
{
    float NoV = saturate( dot( N, V ) );
    float3 R = 2 * dot( V, N ) * N - V;
    float3 PrefilteredColor = PrefilterEnvMap( Roughness, R );
    float2 EnvBRDF = IntegrateBRDF( Roughness, NoV );
    return PrefilteredColor * ( SpecularColor * EnvBRDF.x + EnvBRDF.y );
}
```

很多其他的研究使用了跟unreal一样或接近的的方式来计算lut，其中[Getting More Physical in Call of Duty: Black Ops II](http://blog.selfshadow.com/publications/s2013-shading-course/)更近一步，使用解析拟合的方式来逼近积分。

## Material Model

对于unreal的延迟管线来说，限制参数范围非常重要，这样可以优化gbuffer空间以及texture的存储及获取。
Unreal使用的参数为：

1. Base color
2. Metalic
3. Roughness
4. Cavity

其中Cavity用来进行小尺度（法线尺度上）的投影计算。

最值得注意的是Specular参数的去除，之所以去除，是因为该参数容易引起歧义，美术跟程序很容易设置出不合物理的参数，并且过多参数会限制对效果的把控。

Disney还提供了其他的shading model：

1. Subsurface
2. Anisotropy
3. Clearcoat
4. Sheen

Unreal发表此文章时只提供了subsurface与skin这两种shading model，由于采用纯粹的defer管线，unreal将shading model id存储于gbuffer，然后计算光照时使用动态分支来运行时计算不同shading model。

现在的unreal已经提供了很完备的shading model.

## Experience

Unreal转换pbr的经历验证了，移除speculer，使用金属粗糙度模型并不会影响美术人员的表达，只需要美术人员重新习惯即可。

金属度是否应该是是二值的？是的！金属度就应该是非黑即白的。对于金属度位于01之间的数值应该是使用material layer来表达！即金属覆盖在非金属之上或非金属覆盖在金属之上；

为了保证fortnite的非pbr效果，unreal还是扩充了specular参数，但是并不意味着无法用之前的pbr model来表达非pbr效果，毕竟Disney的动画片已经做出了验证。后续unreal也将会移除specular参数。

## Material layer

material layer大量受益于前面的研究成果，material layer即对材质分层，使得某一材质可以部分覆盖在其他材质之上，从而产生混合材质。

Material layer的优点为：

1. 可重复利用已有资产。
2. 减少单个资产的复杂度。
3. 统一并中心化影响游戏外观的材质，易于美术创作，统一美术风格。

由于unreal节点式的工作流程，使得material layer的集成不会很难。

相比于离线的组装系统，material layer可以提供更高的质量。使用material layer时可以适当提高贴图分辨率，因为material layer的使用场合会非常复杂，除了低频信息，高频信息也会需要。同时因为效率问题也会限制单个材质下material layer的数量，但似乎限制layer的数量并不会影响美术人员的使用。

有两个地方，unreal还没有充分考虑：

1. 由于layer数量的限制，美术会趋近于拆分mesh，来使的每个mesh能获取到足够layer，这样反而会增加draw call，从而影响绘制效率。
2. 对于100%覆盖的区域，使用动态分支是否能优化运行效率，unreal并没有做进一步研究。

## Lighting model

光源模型的建立也会直接影响到着色的物理正确性，unreal主要考虑了两点来改善效果，一个是光源的falloff，另外一个是non-punctual光源，即面光源。

unreal采用了平方衰减的falloff以及使用lumen的物理亮度单位来改善light model，同时考虑到光源范围的问题，falloff的计算公式如下：

$$
falloff = \frac{saturate(1-(distance/lightRadius)^4)^2}{distance^2+1}
$$

## Area lights

面光源在实时渲染中非常重要，如果没有面光源，美术人员将尝试调整roughness来得到面光源下的效果。这是pbr的一大忌讳，材质与光源应该解耦，彼此互不影响。

面光的计算，在离线渲染常使用大量的点光源来进行模拟，通过在光源上均匀采样或者重要性采样来计算。

Unreal对面光源模型的构建有以下要求：

1. 对材质影响的一致性，与其他光源一样，通过材质的diffuse brdf，specular brdf来影响材质的表现。
2. 当solid angel接近0时，light model表现接近于点光源。不能通过修改shading model来达到这一目标。
3. 性能足够好。

## Billboard Reflections

公告板反射是IBL的一种变种，用于离散多光源；具体理论为使用2D的image来存储3D空间下映射在对应Rectangle区域的全部光照，参考[The Technology Behind the DirectX 11 Unreal Engine"Samaritan" Demo](https://docs.unrealengine.com/udk/Three/rsrc/Three/DirectX11Rendering/MartinM_GDC11_DX11_presentation.pdf)。与IBL类似，也需要使用pre-filter来存储不同粗糙度即Cone下的light结果。

尽管Billboard Reflections可以存储任意复杂的面光源信息，但却有以下缺点:

1. 只能在平面上做pre-filtered，因此filte的角度是有限的；
2. 若反射光线没有与plane相机，将没有反射信息；
3. 计算光照时，光照方向是未知的，通常假设为反射方向，即采用的方向；

## Cone Intersection

Cone Intersection的实现不需要进行filter，一个较好的应用版本是[Lighting of Killzone: Shadow Fall](https://www.slideshare.net/guerrillagames/lighting-of-killzone-shadow-fall)中的实现，该方法使用圆锥进行求交计算，将相交区域投射到与圆锥垂直的圆盘上，然后使用多项式近似的NDF来对相交区域进行分段积分，得到近似的光照结果；

这是一个很好的研究方向，不过当前版本并不能满足unreal的需要;即使用Cone Intersection必须是径向对称的，这样会丢失反射在倾斜角度下的拉伸现象，这是反射非常重要的一个特征；此外，与Billboard Reflections一样，计算光照时，光照方向是未知的；

## Specular D Modification

这是Unreal在[The Technology Behind the 3D Graphics and Games Course “Unreal Engine 4 Elemental demo”](https://de45xmedrsdbp.cloudfront.net/Resources/files/The_Technology_Behind_the_Elemental_Demo_16x9-1248544805.pdf)中所采用的技术；背后的理论为：认为光源的分布等同于某一Cone Angle下 $D(h)$ 的分布。光源分布与反射Cone之间的卷积等同于两个Cone角度的相加，从而生成新的cone；

为了达到这种假设，可将 $\alpha$ 转为对应Cone角度，然后加上光源对应的Cone角度，然后再重新转换为 $\alpha^\prime$ ，使用新的 $\alpha^\prime$ 来进行后续运算，即：

$$
\alpha^\prime = saturate(\alpha + \frac{sourceRadius}{3*distance})
$$

尽管这种算法很高效，效果确无法满足Unreal的需求，特别在光滑的材质受到大面积光源照射的情况下，穿帮更加明显；

## Representative Point

对一固定的着色点，可以认为受到面光源的光照，来自于光源表面某一代表性的固定点，这样面光源的光照计算就可以转换为点光源计算；该点的一个合理选择为光源上对该着色点具有最大贡献的点；

对于Phone着色来说，该点就是与反射光线具有最小夹角的点；

使用该方法时，随之带来的便是能量守恒问题，当前此问题并未解决；通过移动光源的发射位置，我们增大了光源的Solid Angle，但却未补充这一附加能量（即面光变点光，但点光的能量确未发生变化）；纠正此问题非常困难，因为此问题会与着色点的Specular 分布有关，即粗糙度；

## Sphere Lights

如果球形光源在着色平面之上，来自球形光源的Irradiance等价于来自于着色平面之上的点光源；这意味着我们只需要计算Specular lighting部分即可；

我们可以光源上计算离反射光线最小角度的点，通过计算离反射光线最近点的方式：

$$
centerToRay = L-(L\cdot r)r \\
closestPoint = L + centerToRay*saturate(\frac{sourceRadius}{|centerToRay|})
$$

其中L表示着色点到光源中心的矢量，r表示反射方向；若反射方向与光源相交，则求出来的是反射光线上距光源中心最近的点；

通过移动光源位置到光源表面，我们实际上会拓宽specular分布至光源对应的cone angle；使用归一化的Phong分布来表示的话，点光源与球光源的分布分别为：

$$
I_point = \frac{p+2}{2\pi} cos^p\phi_r \\
I_{sphere}
\begin{cases}
\frac{p+2}{2\pi} &if\phi_r < \phi_a \\
\frac{p+2}{2\pi} cos^p(\phi_r-\phi_a) &if\phi_r > \phi_a
\end{cases}
$$

这里$\phi_r$表示r与l之间的夹角，$\phi_a$表示球形光源对应的Cone angle；此时点光源是归一化的，积分和为1，球形光很明显将不再归一化；为了近似这种能量的增长，我们使用类似前面提到的 **Specular D Modification** ，对于GGX，归一化因子为$\frac{1}{\pi \alpha^2}$，对于Representative Point的归一化，unreal使用新的拓宽后归一化因子除以原始点光下的归一化因子，即

$$
SphereNormalization = (\frac{\alpha}{\alpha \prime})^2
$$

如此便能得到满足Unreal前面三个要求的计算方法，其中$\alpha\prime$的计算与光源的位置及形状有关，与 **Specular D Modification** 方法中的不太一致，实现需要参考源码，原文中没有提及；

## Tube Lights

首先减少光源半径为0，如此便可认为光源为linear light，linear light上理反射最小angle的近似点为：

$$
t = \frac{(r\cdot L_0)(r\cdot L_d) - (L_0\cdot L_d)}{|L_d|^2-(r\cdot L_d)^2} \\
l = ||L_0+saturate(t)L_d||
$$

为了保证能量守恒，使用类似于针对球形光源的方法，Specular的分布是由光源拓宽的，而linear light是一维的，因此我们可以使用anisotropic GGX的归一化因子$\frac{1}{\pi \alpha_x\alpha_y}$，这里$\alpha_x=\alpha_y=\alpha$，因此Representative Point的归一化为：

$$
LineNormalizetion = \frac{\alpha}{\alpha \prime}
$$

将line与sphere分离，能近似两者的卷积，从而能很好的模拟tube light的光照行为；

基于能量守恒下的Representative Point方法，能很好地模拟简单形状光源，基于此方法，unreal后续可能会加入更多形状的光源；

## 总结

前面介绍了Unreal在Materials、shading以及lighting下如何转型到PBR，此做法已经被证明是非常成功的，通过最新的技术demo，以及Fortnite项目，他能大大提升视觉效果；后续Unreal将会将这些改进带到更多的项目；

## Reference

1. [Real Shading in Unreal Engine 4](https://de45xmedrsdbp.cloudfront.net/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)
