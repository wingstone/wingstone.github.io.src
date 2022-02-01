---
title: 'Pipeline——Rendering Path（渲染路径）'
date: 2020-09-14 14:51:39
categories:
- Unity
tags:
- Unity
- 资源汇总
- 程序化生成
metaAlignment: center
coverMeta: out
---

Rendering Path主要用来指光照计算上的处理方式等；常见的有Forward/Deferred Rendering；以及其改版Forward+、Tiled Based Deferred Rendering、Clustered Shading；
<!--more-->

注意：在render之前，一般还会有一个Application stage，用以在CPU上运行一些必要的前置任务：如碰撞检测、全局加速算法（视锥剔除、遮挡剔除）、物理模拟、动画效果等等；处理完这些后，才能进行高效合理的渲染；

## Forward Rendering

总的来说，前向渲染绘制次数为光源个数乘以物体个数，通常在一个pass中处理一个光源；然后多个pass处理多个光源，并在pass中通过blend add相加得到总的光照效果；

**DC复杂度为O(num(obj)*num(light))**，通常会限制light的数量来减少DC；

更甚者，会将限制光源数量（一般附加光源数量为4），并将所有的光照写进一个Uber Shader，通过传递光照参数来实现；

### Z-Prepass避免overdraw问题

具体来说，在实际渲染之前，加入了一个称之为z prepass的流程，这个流程关闭了color buffer的写入，同时pixel shader极为简单或者索性为空，可以非常快速的执行完毕并且获得场景中的z buffer；紧接着，我们再关闭z buffer的写入，改depth test function为equal。这样就只绘制我们所能看到的像素了（当然只针对于不透明问题）；

## Deferred Rendering

得益于MRT的支持，我们可以发展处延迟渲染，它的核心技术是 **第一阶段** 在绘制物体时将光照所需要的的basecolor、normal、smoothness等信息存储于G-buffer中，而不进行真正的光照；待物体绘制完后， **第二阶段** 再重新使用G-buffer进行光照的计算（及将光照的计算进行推迟）；

传统的延迟渲染在G-Buffer生成之后，会根据光源的形状（light volume），对每个光源执行一次draw call，如果某个像素被light volume覆盖到了，我们就在该像素的位置执行一次当前光源的lighting计算。

需要注意的是，为了防止同一像素被光源正反面计算两次，我们需要在绘制light volume的时候使用单面渲染，如果摄像机在光源内，则需要开启正面剔除，并且将depth test设置为farOrEqual，如果摄像机在光源之外，则开启背面剔除，并且将depth test设置为nearOrEqual。

**DC的复杂度为O(num(obj)+num(light))**；num(obj)为前期绘制物体的数量，num(light)为后期光照时光源的数量；在光照渲染时，同样通过blend add来实现多光源效果的叠加；

一种简单但耗费资源的做法：可以直接将所有光源信息传至一个shader，在这一个shader中进行所有光源的计算与累积，由于是在整个屏幕中进行计算，也就导致会产生多余的光照计算（光源照不到区域也进行了计算）；优点是只有O(1)的复杂度进行光照计算；

另外，G-Buffer除了用于直接照明外，还能够被用于一些间接照明的效果，比如SSAO，SSR；也正是G-Buffer概念的提出，使得近十年来越来越多的算法从world space向screen space的演进；

## Light Pre-Pass

Light Pre-Pass是Deferred Rendering的一个变种，它将整个渲染流程分为**三个阶段**：

1. 只在G-Buffer中存储Z值和Normal值。对比Deferred Render，少了Diffuse Color， Specular Color以及对应位置的材质索引值。
2. 在FS阶段（对应于普通Deferred Rendering的light volume绘制阶段）利用上面的G-Buffer计算出所必须的light properties，比如Normal*LightDir,LightColor,Specular等light properties。将这些计算出的光照进行blend add并存入LightBuffer（就是用来存储light properties的buffer）。
3. 最后将结果送到forward rendering渲染方式计算最后的光照效果；采用Front to Back的绘制顺序，以及前面的LightBuffer进行光照计算；

可以看到光照相关的light properties已经在第二阶段计算过了，第三阶段更多是光照成分的组合，因此又称之为Light Pre-Pass；总体的**DC复杂度为O(num(obj)+num(light)+num(obj))**，分别对应第一二三阶段；

相对于传统的Deferred Render，Light Pre-Pass第三步使用的其实是forward rendering，所以可以对每个mesh设置其材质，这两者是相辅相成的，有利有弊。另一个Light Pre-Pass的优点是在使用MSAA上很有利。

## Tiled Based Deferred Rendering & Forward+

实际上TBDR和Forward+是tiled based方法在forward和deferred上各自的体现，相较于过去的管线，TBDR和Forward+增加了一个 **light culling** 的流程，这个流程把整个屏幕分割成若干个tile（通常每个tile是16*16个pixel），每个tile各自计算出一个单独的**light list**，找出场景中那些对当前tile有贡献的光源。然后对每个tile中的pixel，只需要计算其对应的tile中light list内的光源对该像素的贡献。

### TBDR

**light culling** 后的光照计算过程，是在一个Computer shader中进行计算，针对每个tile，采用Deferred rendering只需累加相应light list的光照即可；**不考虑tile数量，DC复杂度为O(1)**；

由于只读取一次G-buffer，大大减少了带宽的影响；

### Forward+

**light culling** 后的光照计算过程，针对每个tile里的物体，采用Forward rendering在一个shader中累加相应light list的光照即可；**不考虑tile数量，复杂度为O(num(obj))**；

与普通的Forward Rendering相比，其大大减少了DC的数量（因为所有的light都合到一个pass中进行了计算）；

另外，**Z-Prepass**对于Forward+来说是必须的，其大大减弱了Overdraw所带来的性能损耗；毕竟在Forward Rendering+流程中一个pass中进行了大量的light的运算；

## Clustered Shading

TBDR和Forward+使用平面上的tile进行light culling，每一个tile的深度为Z-Near（摄像机近平面）到Z-Far（摄像机远平面）；这样同样会产生问题，即物体对应的像素可能并不会被深度相差太远的光源给照到，因此针对深度上应该再进行一次light culling；

clustered shading就此产生，其给light list的划分增加了一个维度，即depth，它根据view frustum的zmin，zmax把场景进一步根据depth划分成若干个slice（基于指数的划分，通常16个），然后在每个slice上对场景中的所有灯光进行light culling；

在实际shading的时候，每个像素根据自己的depth和screen position，找到对应的depth slice，从3D Texture里拿到offset和num lights，再执行一个num lights次循环；
