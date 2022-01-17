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
移动GPU架构经常被称之为TBDR（Tiled Based Deferred Rendering），我们这里也以TBDR代称；移动架构有TBR与TBDR两种，但实际上两者之间差别没那么大，所以这里拿TBDR来做统一介绍；
<!--more-->

## 移动TBDR架构与桌面IMR架构

### IMR架构

IMR（Immediate Mode Rendering）就如字面意思一样，提交的每个渲染命令都会立即开始执行，并且该渲染命令会在整条流水线中执行完毕后才开始执行下一个渲染命令。

IMR的渲染会存在浪费带宽的情况。例如，当两次渲染有前后遮蔽关系时，IMR模式因为两次draw命令都要执行，因此会存在经过Pixel Shader后的Pixel被Depth test抛弃，这样就浪费了Shader Unit运算能力。不过幸运的是，目前几乎所有的IMR架构的GPU都会提供Early Z的判断方式，一般是在Rasterizer里面对图形的遮蔽关系进行判断，如果需要渲染的图形被遮挡住，那么就直接抛弃该图形而不需要执行Pixel Shader。

IMR的另外一个缺点就是其渲染命令在执行需要随时读写frame buffer，depth buffer和stencil buffer，这带来大量的内存带宽消耗，在移动平台上面访问片外内存是最消耗电量和最耗时的操作。

### TBDR架构

移动端的硬件在设计最开始想到的最重要的问题就是功耗，功耗意味着发热量，意味着耗电量，意味着芯片大小…所以gpu也是把功耗摆在第一位，然而在gpu的渲染过程中，对功耗影响最大的是带宽；

每渲染一帧图像，对FrameBuffer的访问量是惊人的（各种test，blend，再算上MSAA, overdraw等等），通常gpu的onchip memory（也就是SRAM，或者L1 L2 cache）很小，这么大的FrameBuffer要存储在离gpu相对较远的DRAM（显存）上，可以把gpu想象成你家，SRAM想象成小区便利店，DRAM想象成市中心超市，从gpu对framebuffer的访问就相当于一辆货车大量的在你家和市中心之间往返运输，带宽和发热量之巨大是手机上无法接受的。

**TBDR一般的实现策略**是对于cpu过来的commandbuffer，只对他们做vetex process，然后**对vs产生的结果暂时保存**，等待非得刷新整个FrameBuffer的时候，才真正的随这批绘制做光栅化，做tile-based-rendering。什么是非得刷新整个FrameBuffer的时候？比如Swap Back and Front Buffer，glflush，glfinish，glreadpixels，glcopytexiamge，glbitframebuffer，queryingocclusion，unbind the framebuffer。总之是所有gpu觉得不得不把这块FrameData绘制好的时候。

FrameData这个是tbr特有的在gpu绘制时所需的存储数据，在powervr上叫做arguments buffer，在arm上叫做plolygon lists，高通叫bin buffer。

于是移动端的gpu想到了一种化整为零的方法，把巨大的FrameBuffer分解成很多小块，使得每个小块可以被离gpu更近的那个SRAM可以容纳，块的多少取决于你的硬件的SRAM的大小。这样gpu可以分批的一块块的在SRAM上访问framebuffer，一整块都访问好了后整体转移回DRAM上。

### 对比

那么为什么pc不使用tbr，这是因为实际上直接对DRAM上进行读写的速度是最快的，tbdr需要一块块的绘制然后回拷，可以说如果哪一天手机上可以解决带宽产生的功耗问题，或者说sram可以做的足够大了，那么就没有TBDR什么事了。可以简单的认为TBR牺牲了执行效率，但是换来了相对更难解决的带宽功耗。

## TBDR的重要特性

### 关于bin buffer

在生成bin buffer时，会先运行一遍VS，为了得到屏幕坐标从而生成bin buffer；根据bin buffer执行draw call时，会再重新执行VS，PS；因此为了优化第一遍VS的效率，GPU会将VS中与transform不相关的工作分离开来（即SV_POSITION以外输出项的计算），从而优化binning的速度；此项为硬件层优化，无法介入；

### 关于early-z

#### Forward pixel kill

这是我们平时最常了解到的early-z应用形式，在PC上也能看到；是一种在fragment层面上的提前深度测试剔除，即在光栅化后，shading之前进行z的测试，可以节省PS的开销；此项为硬件层优化，无法介入；

#### Hidden surface removal

这是triangle层面上的剔除，执行时间在光栅化之前，若剔除成功就不会执行光栅化及其之后所有的渲染流程，属于比较高效率的优化；此项为硬件层优化，无法介入；

### 关于blending、MSAA、alpha-test

回头看下tbdr的渲染管线，对于一个tile上所有pixel的绘制都是在on-chip的mem上的，只在最后绘制好了才整体回拷给dram。所以我们通常认为会造成大量带宽的操作，例如blending（对framebuffer的读和写），msaa(增加对framebuffer读取的次数)其实在tbdr上反而是非常快速的。（当然msaa除了会造成framebuffer访问增多，还会带来渲染像素的数量增多，这个是tbr没什么优化的）

alpha-test这个东西，他对depth的写入是不能预先确定的，它必须等到pixel shader执行，这导致了alpha-test之后的那些framedata失去了early–z（Hidden surface removal）的机会，也就破坏了TBDR架构中的延迟渲染特性（FrameData必须进行ps处理，不能继续缓存），也就增加了渲染量。

而实际在项目使用中，MSAA仍是非常耗时的，因为对于depth需要多倍的binning及bin buffer，以及对应的内存；alpha blend也没那么高效，因为仍然会增加多倍的overdraw；alpha test消耗可能也没那么高，因为还有fragment层面的early-z存在，以及在PS中提前discard，还可以减少余下ps所带来的开销；

## Reference

1. [MOBILE GRAPHICS 101](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/moving-mobile-graphics)
2. [Mobile HW and Bandwidth](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/moving-mobile-graphics)
3. [针对移动端TBDR架构GPU特性的渲染优化](https://gameinstitute.qq.com/community/detail/123220)
