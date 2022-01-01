---
title: '[Light and Color] Gamma'
date: 2021-06-26 21:11:24
tags: []
published: true
hideInList: false
feature: 
isTop: false
---
# 前言

在图形开发中，总绕不开的一个名词，Gamma，这里会对Gamma进行一个粗暴但是通俗的讲解；便于理清图形开发中理解Gamma的一切；

# 一、Gamma是什么？

Gamma是什么，Gamma FAQ[<sup>[1]</sup>](#refer-anchor)中有解释：在传统的CRT显示器中，屏幕显示的Luminance与电压并不是成正比的关系，而是成一个幂次方的关系，即$L = V^\gamma$，其中的幂就是我们通说所说的Gamma；
![Gamma变换](https://img-blog.csdnimg.cn/20210623234248419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =256x256)
<center>显示器的Gamma变换</center>

# 二、Gamma Correction

## Gamma Correction是什么？

由于显示器所显示的Luminance为正常展示场景亮度的gamma次方（我们称之为Gamma变换），为了使显示器亮度为正常的展示场景亮度，抵消显示器Gamma变换的影响，必须在显示器前对展示场景亮度做 **Gamma变换的逆变换** ，此逆变换我们称之为 **Gamma Correction** ；
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210623234433372.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =256x256)
<center>显示器的Gamma Correction</center>

实际上场景所在的空间，我们常称之为线性空间，即 **Linear Space** ，对应着曲线y=x；经过 **Gamma Correction过后的空间** ，我们称之为Gamma空间，即 **Gamma Space** ，对应着曲线y=x的上半段；

Gamma Correction有时也称 **Encode Gamma** ，从Gamma空间转换到线性空间变换则称之为 **Decode Gamma** ；

## 为什么需要Gamma Correction？

上面已经说到，做Gamma Correction是为了让显示器显示正常的场景亮度，这就是要做Gamma Correction的做主要的原因；

实际上人眼看到现实场景就是线性的，要想让观众觉得显示器里的东西更接近现实，显示器就必须要显示为线性空间；

## 关于Gamma的巧合

实际上， **人眼对亮度的感知也并不是线性的，巧合的是，人眼对亮度的感知曲线搞刚好是显示器Gamma变换的逆变换的曲线（与Gamma矫正类似）**[<sup>[2]</sup>](#refer-anchor) ；

这样导致的结果就是：假如没有Gamma Correction，显示器显示一段均匀变化的亮度，人眼感知到的亮度刚好也是均匀的；因为显示器先做了Gamma变换，人眼感知又做了Gamma逆变换，导致人眼感知到的是均匀的；如下图所示：

![未做Gamma Correction的0-1渐变图](https://img-blog.csdnimg.cn/20210623231605422.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =512x256)
<center>未做Gamma Correction的0-1渐变图</center>

如果我们做了Gamma Correction，这个时候虽然是显示器最终显示的是均匀变化的亮度，但是人眼的感受却是不均匀的；因为先做了Gamma Correction（Gamma逆变换），然后显示器做了Gamma变换，人眼感知又做了Gamma逆变换，导致人眼感知到的是Gamma逆变换后的；如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021062323311715.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =512x256)
<center>做了Gamma Correction的0-1渐变图</center>

做了Gamma Correction之后的渐变图就是人眼真正应该感知到的渐变图：暗部人眼感知到的亮度大于实际亮度，且相差比较大；亮部人眼感知到的亮度与实际亮度基本持平，且相差较小；

之所以人眼有这样的特征，是因为为了能够在夜晚中生存，暗部或被人眼感知上提亮，这样就能够避免夜晚中不可知的危险；

# 三、实际使用的Gamma

## 文件保存中的Gamma

实时上， **我们在电脑屏幕上看到的大部分图片都是处于Gamma空间的** ，尤其是屏幕的截图，屏幕的吸色，得到的都是Gamma空间的颜色，他们经常存储在PNG、TGA、JPG等格式的文件；在显示这些图片时， **他们直接会被送到屏幕缓冲中进行显示** ；由于文件本身为Gamma空间，图片颜色经过显示器的Gamma变换，就会显示为线性空间下的颜色，再进入人眼中，被我们人眼感知；

## 渲染管线中的Gamma

在图形渲染中，有各种各样的环节，包括PBR Shading、后期Color Grading、后期Tone Mapping、AA等等；同一个环节内也可能包含各种各样的模块与环节，比如PBR Shading环节就包括贴图读取、BRDF相关矢量计算、BRDF计算等（不管这些是在VS阶段、还是PS阶段）；
在这些环节中大部分的计算会与颜色相关，说到颜色也就伴随着Gamma空间与线性空间；那么在整个渲染管线中，应给如何处理Gamma问题呢？
关于Gamma的处理，首先确认的是，显示器显示的空间应该是线性空间（前面的为什么需要Gamma Correction已经提到），所以输送到屏幕缓冲的颜色应该为Gamma空间，为了抵消屏幕的影响；另外要注意的是，我们关于颜色的光照计算或者其他计算应该在线性空间（除非一些必须在Gamma空间或其他空间的计算），以为线性空间的颜色才是物体本身的颜色，在线性空间下进行光照或其他计算才是物理正确的，否则都是物理不正确的；
所以 **整体的流程** 应该是：

1. 读取贴图颜色，位于[Gamma Space](#文件保存中的Gamma)；
2. Decode Gamma到Linear Space，然后进行光照或其他关于颜色的计算；
3. Encode Gamma到Gamma Space（Gamma Correction），然后写入到屏幕缓存；
4. 显示器进行Gamma变换，显示为Linear Space下的颜色；

下图为RTR中在现实过程中Gamma的处理[<sup>[3]</sup>](#refer-anchor)，与我们之前所理解的一致；
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210626165646704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =512x256)
<center>Gamma process in display</center>

## Gamma使用不当会引起的问题

事实上，早期人们还没有意识到Gamma的问题，以至于很多图形上的操作都是在Gamma空间下进行的，以至于出现了很多画面上的问题，包括Light fall-off、Linear intensity response、Linear and gamma blending等[<sup>[4]</sup>](#refer-anchor)；分别如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021062618200573.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =480x256)
<center>Light fall-off</center>

Light fall-off影响的为光照沿平面朝向的衰减，其实是就是计算该点的irridience，在diffuse上会有明显的感受；可以看到Gamma Space下亮度分布更均匀，Linear space下亮度分布更接近[人眼感知](#关于Gamma的巧合)；

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210626181743896.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =450x256)
<center>Linear intensity response</center>

Gamma空间下，不同光强产生的影响会更加反直觉，会非常容易产生过曝的问题，以至于在项目中美术打光很难控制光照强度；

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210626182410735.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODE3NzAw,size_16,color_FFFFFF,t_70#pic_center =256x256)
<center>Linear and gamma blending</center>

Gamma空间下的Blend则会引起色相偏移问题（插值、叠加或其他Blend方式），这在工程中对美术来说是很不友好的，特别是游戏中的特效，受此影响最为过大；在RTR4中则还提到了叠加引起的亮度感知错误问题，在5.6章节，有兴趣的可以看一下；
![另外](https://img-blog.csdnimg.cn/2021062618520422.png#pic_center =500x100)
<center>Ropping</center>

Ropping现象则是因为没有经过Gamma Correction引起的直线弯曲效应；不过需要在使用MSAA或者SSAA的情况下才能看到，上图就是开8xAA情况下出现的问题；

# 总结

本文主要介绍了图形学中Gamma的由来，以及图形开发中与Gamma相关的场景；开发者们需要注意Gamma的存在，并通过正确的线性工作流来避免Gamma引起显示不正确问题；

<div id="refer-anchor"></div>

# References

[1] [Gamma FAQ](http://poynton.ca/notes/colour_and_gamma/GammaFAQ.html)
[2] [What are Color Spaces, Color Profiles and Gamma Correction?](https://www.wigglepixel.nl/en/blog/what-are-color-spaces-color-profiles-and-gamma-correction)
[3] [realtimerendering](http://www.realtimerendering.com/)
[4] [Linear or gamma workflow](https://docs.unity3d.com/Manual/LinearRendering-LinearOrGammaWorkflow.html#:~:text=Linear%20or%20gamma%20workflow%20The%20Unity%20Editor%20offers,of%20a%20mesh%20to%20give%20it%20visual%20detail.)
