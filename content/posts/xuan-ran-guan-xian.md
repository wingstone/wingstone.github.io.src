---
title: 'Pipeline——Render Pipeline/Path（渲染管线/路径）'
date: 2020-09-14 14:51:39
tags: [图形学,pipeline]
published: true
hideInList: false
feature: 
isTop: false
---

Render Path称之为渲染路径更为合适，实际上指渲染一帧所要走的流程，这个流程主要用来处理光照，以及后处理等；常见的有Forward/Deferred Rendering；以及其改版Forward+、Tiled Based Deferred Rendering、Clustered Shading；以及更灵活的Frame Graph（寒霜引擎）、SRP（Unity引擎）；

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

## Unity Scriptable Render Pipeline（SRP可编程渲染管线）

Unity SRP实际上是对底层渲染API进行了封装，并且结合unity内部的渲染模块，提供了一个相对高级的可编程化的渲染管线；用户可以使用SRP编写适合自己产品的Render Pipeline；不过要想自定义SRP，必须要对已有的管线有深入了解，才能编写出高效的管线；

在Unity SRP出现之前，Unity提供的是不可更改的standard built-in pipeline；Unity SRP出现后，Unity还提供了两个已经相对标准且开源的基于SRP的管线，**URP与HDRP**；URP主要侧重于通用平台下的渲染，支持多平台；HDRP主要侧重高端平台下的渲染（大量使用了Computer Shader），普通平台支持不了；

## Frame Graph

随着时代的发展，内置固化的渲染管线架构已经越来越不能满足行业发展的需求，具体表现在以下几个方面：

1. 多样化的游戏画面表现需求：有的写实，有的卡通，有的更加风格化，这就要求引擎的渲染管线能具有相应的可调整的能力；
2. 不同平台的软硬件能力差异：比如 PC、主机硬件能力更强，需要有更高端的画面表现，试图通过一种渲染流程来充分发挥每个平台的优势以及消除劣势，是一件非常困难的事情；
3. Rendering Path 的不断进化：即前面介绍的各种render path；
4. 新的渲染技术：近年来的 VR\AR\XR 渲染技术对渲染管线提出了特定的需求，一套管线已经很难做到完全兼容这些渲染机制。在 2018 年微软发布了革命性的 DirectX RayTracing，这更是完全不同Rendering\Computing 机制。

通过上述几点，说明传统的固定功能的渲染管线架构已经不再适合未来游戏发展的需求，如果重新设计引擎的渲染管线，必须要考虑到渲染管线的可扩展性、可配置性、甚至是 Data-Driven 的。

**Frostbite：Frame Graph（FG）**就此产生，FG 由** RenderPass** 以及其依赖的 **Resource **组成。RenderPass 定义了一个完整的渲染操作，Resource 包括了 RenderPass 使用的 PSO、Texture、RenderTarget、ConstantBuffer、Shader 等资源。每个 RenderPass 都有 Input 和 Output 资源，这样 RenderPass 和 Resource 就形成了有向非循环图（DAG）结构，因为描述的是引擎在一帧内的渲染流程，所以称之为 Frame Graph。

更详细的Frame Graph接受看[这里](https://www.cnblogs.com/username/p/8497150.html)；

在 RG 中，每个 RenderPass（RP） 包含三个重要阶段：

1. Setup 阶段，主要用于定义输入和输出所使用的资源，比如 RenderTarget、Buffer 等。
2. Compile 阶段，根据 Setup 所定义的资源，来决定真正需要执行的 RP 执行路径图，并且裁剪掉不需要执行的 RP 依赖资源。
3. Execute 阶段，是真正执行 RP 渲染逻辑的阶段。在这个阶段可以直接调用渲染 API 将 Render Command 和 GPU 资源提交到渲染设备中，完成真正的渲染。

Setup 一般只需要执行一次，Execute 每帧都需要执行，Compile 可根据情况执行多次，比如某个 RP 根据配置动态变化，这时需要重新 Compile，以获得变化后的 RP 执行路径图。