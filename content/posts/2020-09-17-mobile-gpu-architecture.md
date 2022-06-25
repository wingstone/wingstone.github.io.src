---
title: '移动GPU架构'
date: 2020-09-17 13:30:09
categories:
- GPU
tags: 
- 总结
- Mobile
- GPU
---
移动GPU架构经常被称之为TBR（Tiled Based Rendering），我们这里也以TBR代称；
<!--more-->

## 移动TBR架构与桌面IMR架构

### IMR架构

IMR（Immediate Mode Rendering）就如字面意思一样，提交的每个渲染命令都会立即开始执行，并且该渲染命令会在整条流水线中执行完毕后才开始执行下一个渲染命令。

IMR的渲染会存在浪费带宽的情况。例如，当两次渲染有前后遮蔽关系时，IMR模式因为两次draw命令都要执行，因此会存在经过Pixel Shader后的Pixel被Depth test抛弃，这样就浪费了Shader Unit运算能力。不过幸运的是，目前几乎所有的IMR架构的GPU都会提供Early Z的判断方式，一般是在Rasterizer里面对图形的遮蔽关系进行判断，如果需要渲染的图形被遮挡住，那么就直接抛弃该图形而不需要执行Pixel Shader。

IMR的另外一个缺点就是其渲染命令在执行需要随时读写frame buffer，depth buffer和stencil buffer，这带来大量的内存带宽消耗，在移动平台上面访问片外内存是最消耗电量和最耗时的操作。

### TBR架构

移动端的硬件在设计最开始想到的最重要的问题就是功耗，功耗意味着发热量，意味着耗电量，意味着芯片大小…所以gpu也是把功耗摆在第一位，然而在gpu的渲染过程中，对功耗影响最大的是带宽；

TBR的思想是将将屏幕划分成tile，然后即可在GPU上的low-latency memory（GMEM）上进行逐tile渲染，在一个tile渲染完成后，将tile对应的color buffer、depth buffer拷贝到system memory；由于在tile上渲染，避免了逐像素对system memory的大量读写，除了最后的拷贝操作，因此节省了大量带宽；具体可参考[Tile-based rendering](https://developer.qualcomm.com/sites/default/files/docs/adreno-gpu/developer-guide/gpu/overview.html#tile-based-rendering)；

### 对比

那么为什么pc不使用tbr，这是因为实际上直接对DRAM上进行读写的速度是最快的，tbr需要一块块的绘制然后回拷，可以说如果哪一天手机上可以解决带宽产生的功耗问题，或者说sram可以做的足够大了，那么就没有TBR什么事了。可以简单的认为TBR牺牲了执行效率，但是换来了相对更难解决的带宽功耗。

## TBR的重要特性

### 关于bin buffer

在生成bin buffer时，会先运行一遍VS，为了得到屏幕坐标从而生成bin buffer，bin buffer中记录了屏幕中各个tile所关联的triangles；根据bin buffer执行draw call时，会再重新执行VS，PS；因此为了优化第一遍VS的效率，GPU会将VS中与transform不相关的工作分离开来（即SV_POSITION以外输出项的计算），从而优化binning的速度；此项为硬件层优化，无法介入；

> 此处内容从一次高通的分享中了解到，但暂未在某一文档中看到；

### 关于early-z

#### Forward pixel kill

这是我们平时最常了解到的early-z应用形式，在PC上也能看到；是一种在fragment层面上的提前深度测试剔除，即在光栅化后，shading之前进行z的测试，可以节省PS的开销；此项为硬件层优化，无法介入；

#### Hidden surface removal

这是triangle层面上的剔除，执行时间在光栅化之前，若剔除成功就不会执行光栅化及其之后所有的渲染流程，属于比较高效率的优化；此项为硬件层优化，无法介入；

### 关于blending、MSAA、alpha-test

回头看下tbr的渲染管线，对于一个tile上所有pixel的绘制都是在on-chip的mem上的，只在最后绘制好了才整体回拷给dram。所以我们通常认为会造成大量带宽的操作，例如blending（对framebuffer的读和写），msaa(增加对framebuffer读取的次数)其实在tbr上反而是非常快速的。（当然msaa除了会造成framebuffer访问增多，还会带来渲染像素的数量增多，这个是tbr没什么优化的）

alpha-test这个东西，他对depth的写入是不能预先确定的，它必须等到pixel shader执行，这导致了alpha-test之后的那些framedata失去了early–z（Hidden surface removal）的机会，也就破坏了TBR架构中的延迟渲染特性（FrameData必须进行ps处理，不能继续缓存），也就增加了渲染量。

而实际在项目使用中，MSAA仍是非常耗时的，因为对于depth需要多倍的binning及bin buffer，以及对应的内存；alpha blend也没那么高效，因为仍然会增加多倍的overdraw；alpha test消耗可能也没那么高，因为还有fragment层面的early-z存在，以及在PS中提前discard，还可以减少余下ps所带来的开销；

## 高通特有的特性

这一部分的更多细节可以直接参考高通的文档[Game Developer Guides](https://developer.qualcomm.com/sites/default/files/docs/adreno-gpu/developer-guide/index.html)；

1. FlexRender™ technology (Hybrid Deferred and Direct Rendering mode)：高通所单独开发的特性，可以支持tbr与imr模式的混合使用，从而兼顾两者的优点；如果使用snapdragon profiler来分析unity在高通芯片上渲染的一帧，就能发现在场景渲染阶段，高通使用tbr来渲染，在后处理阶段，高通使用imr来渲染，从而使得功耗及效率达到最优；
2. Low Resolution Z pass：此功能在Adreno 5X (A5X)处理器上加入，并且该功能是与渲染顺序无关的功能；该功能可以在binning pass阶段来构建低分辨率下的depth buffer，随后在rendering pass使用LRZ来进行高效且与与顺序无关的depth rejection；更详细解释可参考[Low Resolution Z Buffer support on Turnip](https://blogs.igalia.com/siglesias/2021/04/19/low-resolution-z-buffer-support-on-turnip/)；

## mali特有的特性

在arm的开发者文档上能了解到更多的内容[mali GPU Architectures](https://developer.arm.com/Architectures#aq=%40navigationhierarchiescategories%3D%3D%22Architecture%20products%22%20AND%20%40navigationhierarchiescontenttype%3D%3D%22Product%20Information%22&numberOfResults=48&f[navigationhierarchiesprocessortype]=GPU%20Architectures)；整个mali gpu家族对应的型号、特性、架构可以在[About the Mali GPU hardware families](https://developer.arm.com/documentation/100587/1-1/The-Mali-GPU-Hardware-Families/About-the-Mali-GPU-hardware-families?lang=en)了解到；

在文档上并未找到更多的关于mali特有的架构特性，但在文档中介绍TBR时，详细介绍了TBR所带来的的优势与劣势[Tile-based GPUs](https://developer.arm.com/documentation/102662/0100?lang=en)；
1. 带宽优势，除了上面提到的带宽优势，mali在拷贝tile color至system mem时，会启用‘Cyclic Redundancy Check' (CRC)检测，如果tile color与system color并无变化，则不需要进行mem的拷贝，从而节省带宽；在很多ui渲染，以及休闲游戏，此特性能带来很大的带宽收益；在进行mem拷贝时，mali会启用 ‘Arm Frame Buffer Compression' (AFBC)的特性，用来进行color data的无损压缩，从而节省带宽；可以看出，在节省带宽上，mali做了很多额外的工作；
2. 算法优势，TBR的使用，是的很多高带宽的算法得以高效实现（因为tile的带宽优势），如MSAA与MRT；对于MSAA来说，在从tile写入system mem的过程中即可完成resolve的操作；对于MRT，在defer lighting时读取frame buffer可以受益（看原文的意思，应该是因为以tile的形式读取导致的）；
3. 由于tbr算法特性，必须要存储geometry的输出（per vertex data）与tile intermediate  state至system mem，因此在geometry pass与fragment pass之间会产生额外开销，必须要在此额外开销与tbr所带来的的带宽优势取得平衡，才能带来受益；例如tessellation技术就是完全为imr架构所设计的，在imr架构中会将额外的geometry data存至 on-chip FIFO buffer，而不是system mem；


## Reference

1. [MOBILE GRAPHICS 101](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/moving-mobile-graphics)
2. [Mobile HW and Bandwidth](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/moving-mobile-graphics)
3. [针对移动端TBDR架构GPU特性的渲染优化](https://gameinstitute.qq.com/community/detail/123220)
