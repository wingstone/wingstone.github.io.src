---
title: 'Color Space'
date: 2021-06-26 23:14:15
categories:
- Light&Color
tags: 
- 总结
- Color Space
---

颜色空间作为渲染领域的理论基础，掌握后才能保证渲染显示结果的正确性；这里介绍了基础的颜色空间知识，并介绍一些常用的颜色空间；
<!--more-->

## Physical Basis of Color（颜色的物理基础）

说道颜色，首先得从光说起，因为光具有真实的物理属性；

- 光是由不同波长的光照成分组成的，人眼能看到的光的范围称之为可见光谱；
- Spectral Power Distribution (SPD)为谱功率密度，表示一束光在不同波长下的功率分布；常用来测量真实的光；
- SPD具有线性叠加性；

说完光，再来说颜色，颜色是光在人眼感知下的结果；并不是光的通用属性，与人眼具有强关联性；

## Biological Basis of Color（颜色的生物基础）

人眼球中分布着Rods cell（杆状细胞，用来识别亮度），Cone cells（锥状细胞，用来识别颜色）；
锥状细胞分为三种类型：S、M以及L，三种细胞对于不同波长的光，具有不同的反映峰值，各拥有各的spectral response curve；

## Tristimulus Theory of Color（颜色的三色理论）

前面已知不同锥状细胞具有不同的反映曲线，人的大脑处理的实际上是三种锥状细胞处理后输出的信号；即输入SPD与三种Response Function积分后得到的三个分量（S,M,L）；

## Luminosity Function（亮度函数）

锥状细胞感受颜色，杆状细胞感受强度；同样不同的波长，杆状细胞感受出来的强度是不同的，感官亮度可以表示为Luminosity Function亮度随波长变化的函数；

### Metamerism（同色异谱）

由于是积分产生的结果，那么不同的输入积分后就有可能产生同样的结果；即人眼看到的同样的颜色，可能是由不同的SPD产生；

## Color Reproduction / Matching（颜色匹配）

同色异谱机制使得颜色匹配得以进行；Color Matching首先指定三盏光（每盏拥有固定SPD），调整三盏光的强度，使得混合后的颜色能匹配制定测试光，三盏光得到的强度由RGB组成，即为测试光光强；
有时Color Matching的测试光需要进行补光，此时获取的RGB值可为负值；

### CIE RGB Color Matching Experiment（CIE颜色匹配实验）

CIE颜色匹配实验是CIE所推出的颜色匹配标准；

- CIE使用red=700nm, green=546.1nm, blue=435.8nm三种波长的纯色光来进行匹配；
- 对可见波长的纯色光进行匹配后，得到了三条颜色匹配曲线，称之为Color Matching Curves或Color Matching Function；
- 有了颜色匹配曲线后，对于任意的SPD，只需将其与三条曲线进行积分，即可得到CIE rgb值，该值所在的空间，称之为CIE RGB空间；

## Color Space（颜色空间）

### LMS Space

此处LMS即为前面所说的眼睛内部的三种锥形细胞，以三种细胞的response curve作为积分曲线，积分后得到的颜色值位于LMS 空间；Unity中的白平衡既是在此空间下进行计算；

### CIE RGB Space

前面提到将SPD与CIE color matching curves进行积分后即可得到CIE RGB空间下的值；

### CIE XYZ Space（1931）

同样CIE RGB Space里的值会存在负值，由于匹配主光设定的限制；
为了避免负值的不变，CIE将RGB color matching curves经过仿射变换为XYZ color matching curves，转换后的curves不含有负值，这样积分得到的值就不会存在负值；积分值所在的空间称之为CIE XYZ Space；

- XYZ空间使用的Color Matching Curves是经过精心设置的，其中的Y坐标的curve刚好为人眼的Luminosity Function（亮度函数），这样XYZ空间下的Y坐标刚好就代表着亮度；
- 与CIE RGB空间相比，RGB空间的三个轴（前面所提的单色光）具有实际的物理波长，而XYZ空间的三个轴并没有物理意义，也就没有实际的物理颜色与波长；

### CIE LUV and LAB（1976）

CIE 1931 XYZ具有一个很大的缺点，就是其上的颜色并不是均匀显示的，即相同间隔的颜色，在色度图上所显示的距离最多能相差20倍；为了解决这个问题CIE LUV就被提了出来，其将XYZ重新进行变换，能将最大相差倍数缩小为4倍；

### xyY Space

由于色度图（下个小标题介绍）的方便性，使用色度图中的xy可以很方便的表示颜色的色相，但却丢失了亮度，因此加上CIE XYZ空间中Y值即可重建原来的颜色；
xyY也因为其方便性而经常使用，所以这里把xyY单独列出来；

### sRGB

sRGB是如今实时渲染中使用最广泛的颜色空间，广泛使用在PC以及TV上；值得一提的是，sRGB与Rec. 709使用相同的Primaries lights以及白点，其中白点使用D65；
> 注意：这里sRGB指的是linear light space，并不考虑Gamma等相关因素的下nonlinear color space；相关知识参考[这里](https://blog.csdn.net/candycat1992/article/details/46228771)还有[这里](https://zhuanlan.zhihu.com/p/36581276);

### LogC

Log Color Space是基于场景的编码颜色空间，我们经常使用的sRGB这种则是基于显示的编码颜色空间；不同的厂商有不同的Log编码方式，LogC是ARRI用来记录颜色的空间；参考[Understanding Log and Color Space In Compositing](https://www.rocketstock.com/blog/tips-for-log-color-space-compositing/)以及[LogC](https://www.arri.com/en/learn-help/learn-help-camera-system/camera-workflow/image-science/log-c)；
LogC最大的优点是维持更多的动态范围信息，甚至是超过人眼所能感受的范围；由于其通过对数方式编码，存储范围远远大于一般的颜色编码方式；
由于LogC特性，常在LogC空间下进行对比度的计算，对比度的计算可能会导致负数、以及远超1的数，这些都可以在LogC空间下进行保存，使用再从LogC空间解码到原来的空间即可；

### HSV

H为色相（Hue）、S为饱和度（Saturation）、V为亮度（Value）；HSV也被称作为HSB（Brightness），它是RGB空间的非线性转换，并不是独立的颜色空间，因此没有RGB Primaries以及白点的定义[参考这里](https://psychology.wikia.org/wiki/HSV_color_space)；

### ACES

ACES并不是一个单独的颜色空间，而是一个系统，包含了颜色空间的一整套系统；ACES的存在就是为了标准化整个颜色的工作流；其内部包含了各颜色空间到ACES空间的转换，ACES内部还定义了ACES2065-1颜色空间（常称为AP0），该空间包含了整个色度图的范围，采用D60作为white point；还定义了ACEScg颜色空间，该空间常作为compositing与cg所使用的空间（常称为AP1），虽不如AP0，但也涵盖了大量的色域，采用D60作为white point；另外还有一个ACEScc，ACEScc与ACEScg基本一致，除了其使用了log gamma的形式，常用在grading中；ACES最后还定义了ACES空间至屏幕显示空间的转换，常有RRT与ODT组成，RRT与ODT内部包含的内容比想象中的要复杂的多，不仅考虑了Gamm，tonemapping，还有很多其他的因素；

## CIE chromaticity diagram(CIE色度图)

直接在CIE XYZ空间上工作有众多不便；常用的做法是将XYZ投影到x+y+z=1的平面；即将XYZ进行归一化得到xyz值，这样只需要记录xy值（范围为0-1），z值为1-x-y；xy值所在的空间被称之为CIE色度图（特指CIE 1931 xy chromaticity diagram）；

- 色度图包含了人眼所能看到的所有颜色，当然不包括强度；
- 实时上在色度图上丢失的是亮度信息Y，而xy决定了所有的颜色；
- 色度图为马蹄形；马蹄形的边沿代表整个色谱，是单色（单一波长），马蹄内部的颜色是混合色；
- white point位于色度图的中心某处，具体位置依靠于所使用的光源；
- 色度图还具有很多很好的性质，色相与饱和度也可以从色度图中找出，具体看一查看这里[Colorimetry](https://www.cg.tuwien.ac.at/research/theses/matkovic/node14.html)；

## Gamut(色域)

色度图包含了了所有的颜色，色度图中任意三点（Color Primaries）所组成的三角形就是一个色域（由于色度图的形式，三点进行插值可以得到三角形内的所有颜色）；我们平时使用的sRGB就是其中的一个色域；

- 色域只包含所能表示的颜色，不能表示颜色的强度；
- 由于色度图是CIE XYZ空间的投影，所以CIE RGB空间的色域也能在色度图中进行表示，CIE  RGB在色度图中所显示的色域同样也为一个三角形；
- CIE XYZ是由CIE RGB转换而来的的，为了CIE RGB的色域在色度图中反而是三角形，会丢失一些颜色呢？就是因为CIE RGB包含了负数，这些负数在色度图中丢失了；参考这里[Why is there a difference between the CIE XYZ colour gamut vs CIE RGB](https://computergraphics.stackexchange.com/questions/10114/why-is-there-a-difference-between-the-cie-xyz-colour-gamut-vs-cie-rgb/10159#10159?newreg=7627228e6b9c4129811bd19902710039);
- 各个颜色空间都有自己的色域；

## Color Temperature（色温）

谈到色温就要谈到黑体辐射；黑体在物理上被认为是一种物质，它能吸收打到它身上所有的光线，同时会随着自身温度的变化，而释放出固定的光谱；

- Black Body是一个理想化的物理物体，之所以叫黑体，是以为它会吸收所有的光线；
- 由于对于黑体来说，温度与黑体所释放出来的光谱是固定的，我们常用色温来指定相应的光谱；
- 黑体辐射的光谱与自然光是非常相似的，因此常用色温来指定阳光的颜色；
- 色温有时也被用来指定白点，D65就是一个例子；

## White Point（白点）

白点，定义的为一个色度，定义了白色在primaries light下的比值，具有一系列颜色与强度无关；某个光照环境下的白点，定义为白色物体在该环境下的色度；

- 最常用的是白点是D65，定义为色温为6500K下的daylight光照环境；

## Define a Color Space（定义一个颜色空间）

前面所说，色域已经给出了一个颜色空间所能表示的所有颜色，唯独丢失了强度范围，primaries lights给我们提供了矢量的方向信息，却没有没有提供矢量的相对比例关系；white point这时就起到了这个作用，白点确定了primaries lights之间的比例关系，之后再将白点亮度（Y坐标）缩放到1，即可确定给定primaries lights下所有颜色；

- Three Primaries Lights；
- White Point；

## Transfer Between Color Space（转换颜色空间）

由于每个颜色空间的色域都在XYZ空间上进行了标定（由primaries lights来决定），同时颜色空间下的White Point也已经指定，这样有颜色空间转换到XYZ空间的矩阵就可以确定；
该矩阵具体由White Point在色度图上的xy坐标，primaries lights的xy坐标计算得来；
同理，另外一个颜色空间转换至XYZ空间下的矩阵也可以确定，以XYZ空间为媒介，就可以在不同的颜色空间中进行转换；

> 注意：实际上真正的物理渲染应该使用SPD来进行光照的计算与积分，即入射光使用SPD，brdf使用Spectral Reflectance Curve（）SRC，两者的积分与使用RGB直接计算得到结果会存在差异；但多数情况下，这种差异可以忽略，特别是在实时渲染；
> 正确的渲染过程应该是光照的整个积分与计算都是用SPD和SRC来进行，只在最后输出的时候才转换成RGB显示；

## References

1. [Color FAQ](http://poynton.ca/notes/colour_and_gamma/ColorFAQ.html)
2. [Colorimetry](https://www.cg.tuwien.ac.at/research/theses/matkovic/node14.html)
3. [brucelindbloom](http://www.brucelindbloom.com/)
4. [How to Choose the Right Video Color Space](https://www.richardlackey.com/choosing-video-color-space/)
5. [What Is A Video Color Space?](https://www.richardlackey.com/what-is-a-video-color-space/)
6. [hitchhikers guide to digital color](https://hg2dc.com/)
7. [COLORIMETRY](https://color2.psych.upenn.edu/brainard/papers/Brainard_Stockman_Colorimetry.pdf)
8. [HDR in Call of Duty](https://research.activision.com/publications/archives/hdr-in-call-of-duty)
9. [Colour & Vision Research laboratory](http://www.cvrl.org/)
10. [White Point](https://en.wikipedia.org/wiki/White_point)
11. [Black Body](https://en.wikipedia.org/wiki/Black_body)
12. [AN IN-DEPTH LOOK AT ASC-CDL BASED COLOR CONTROLS](https://pomfort.com/article/an-in-depth-look-at-asc-cdl-based-color-controls/)
13. [Cinematic Color](https://cinematiccolor.org/)
14. [Lift Gamma Gain](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@7.1/manual/Post-Processing-Lift-Gamma-Gain.html)
15. [ACES pdf](https://z-fx.nl/ColorspACES.pdf)
16. [ACES Space](https://acescolorspace.com/)
17. [Color system](https://chrisbrejon.com/cg-cinematography/chapter-1-5-academy-color-encoding-system-aces/)
