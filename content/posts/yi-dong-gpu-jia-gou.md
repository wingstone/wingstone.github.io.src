---
title: '移动GPU架构'
date: 2020-09-17 13:30:09
tags: [图形学,GPU]
published: true
hideInList: false
feature: 
isTop: false
---
移动GPU架构经常被称之为TBDR（Tiled Based Deferred Rendering），我们这里也以TBDR代称；实际上移动架构有TBR与TBDR两种，为什么都被称之为TBDR，可以看这篇[文章](https://zhuanlan.zhihu.com/p/112120206)；我们这里使用TBDR来指整个移动GPU架构都包含的TBR特点；
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

FrameData这个是tbr特有的在gpu绘制时所需的存储数据，在powervr上叫做arguments buffer，在arm上叫做plolygon lists。

于是移动端的gpu想到了一种化整为零的方法，把巨大的FrameBuffer分解成很多小块，使得每个小块可以被离gpu更近的那个SRAM可以容纳，块的多少取决于你的硬件的SRAM的大小。这样gpu可以分批的一块块的在SRAM上访问framebuffer，一整块都访问好了后整体转移回DRAM上。

### 对比

那么为什么pc不使用tbr，这是因为实际上直接对DRAM上进行读写的速度是最快的，tbdr需要一块块的绘制然后回拷，可以说如果哪一天手机上可以解决带宽产生的功耗问题，或者说sram可以做的足够大了，那么就没有TBDR什么事了。可以简单的认为TBR牺牲了执行效率，但是换来了相对更难解决的带宽功耗。

## TBDR的重要特性

### 关于early-z
因为tbdr有framedata队列，很多gpu会很聪明的尽量筛去不需要绘制的framedata。所以在tbdr上earlyz，或者stencil test这些是非常有益处的。例如你定义了一个stencil，gpu有可能在对framedata处理的过程中就筛掉了那些不能通过stencil的drawcall了。或者通过scissor test可能一整块tile都不需要绘制。

### blending和MSAA的效率其实很高，alpha-test效率很低
回头看下tbdr的渲染管线，对于一个tile上所有pixel的绘制都是在on-chip的mem上的，只在最后绘制好了才整体回拷给dram。所以我们通常认为会造成大量带宽的操作，例如blending（对framebuffer的读和写），msaa(增加对framebuffer读取的次数)其实在tbdr上反而是非常快速的。（当然msaa除了会造成framebuffer访问增多，还会带来渲染像素的数量增多，这个是tbr没什么优化的）

alpha-test这个东西，他对depth的写入是不能预先确定的，它必须等到pixel shader执行，这导致了alpha-test之后的那些framedata失去了early–z的机会，也就破坏了TBDR架构中的延迟渲染特性（FrameData必须进行ps处理，不能继续缓存），也就增加了渲染量。

## 移动TBDR架构与Render Pipeline中TBDR的区别

Render Pipeline中TBDR可以参考[这里](xuan-ran-guan-xian)；RP中的TBDR指的光照渲染流程中，延迟光照的一种；为了减少DC，提高硬件利用效率；**这里的延迟主要指**：PS阶段光照的计算不立即计算，而是渲染到G-buffer中进行缓存，到最后使用的新的DC来使用G-buffer进行光照的计算（主要用于多光源下的渲染处理）；

移动TBDR架构指的是GPU所采用的的一种渲染架构，是GPU硬件上的延迟渲染流程的硬件实现；而且**这里的延迟渲染主要指**：VS到PS之间有一个硬件的framedata队列，来延迟PS的处理；

## Reference

[移动架构浅析](https://gameinstitute.qq.com/community/detail/103959)
[移动设备GPU架构知识汇总](https://zhuanlan.zhihu.com/p/112120206)
[针对移动端TBDR架构GPU特性的渲染优化](https://gameinstitute.qq.com/community/detail/123220)