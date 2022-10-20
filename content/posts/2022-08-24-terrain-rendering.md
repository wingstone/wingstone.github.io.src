---
title: 'terrain rendering'
author: wingstone
date: 2022-08-24
categories:
- contents
metaAlignment: center
coverMeta: out
---

地形渲染相关记录

<!--more-->

## Mesh

### 基于四叉树的mesh

可参考文章为[terrain rendering in far cry](https://www.gdcvault.com/play/1025480/Terrain-Rendering-in-Far-Cry)与[大世界GPU Driven地形入门](https://zhuanlan.zhihu.com/p/388844386)

1. 地形结构的四叉树生成，由于每个节点所使用的mesh都是相同的，一般为16x16的patch，不同层级的mesh只是scale与位置不一样，从而形成天然的lod结构；
2. 每个节点可以只存储地形节点的boundingbox、xz位置、以及lod；lod的确定可以使用到相机的距离，也可以使用其他影响因子；
3. 逐个遍历节点，进行四叉树的剔除，view culling、back culing、hiz culling；可GPU化；
4. 生成整个地形的lodmap图，用于生成时，根据lod的差值来调整接缝问题；可GPU化；
5. 逐节点渲染，根据世界坐标的xz生成uv，采样高度图，调整高度从而生成地形；生成节点高度时，需要根据与相邻lod的差值，进行stitch处理；可以在cpu上做，也可以再vs上做；

### 基于geometry clipmap的mesh生成

可参考文章为[Terrain Rendering Using GPU-Based Geometry Clipmaps](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-2-terrain-rendering-using-gpu-based-geometry)、[Terrain Rendering with Geometry Clipmaps](https://arm-software.github.io/opengl-es-sdk-for-android/terrain.html)、[Geometry-Clipmap-Tutorial-in-OpenGL](https://github.com/sp4cerat/Geometry-Clipmap-Tutorial-in-OpenGL)

整个地形所使用的mesh，除了中心的四方形mesh，其他所有的mesh都是采用相同的环形mesh嵌套形成，同样具有天然的lod形式，越外层的环形mesh，其scale越大；

Nvidia在上面的文章中采用了基于GPU实现的方法；有几个值得提到的点：

1. 环形网格进行了拆分，使得环形mesh节点可以进行剔除；
2. 接缝，使用插值随后blend的方式进行处理；
3. 高度图精度不够问题，使用upsampling技术，即曲面细分技术进行解决；

不过从其他地方看到的实现方式来看，完全可以使用同样的patch，采用clipmap的缩放方式，来拼接成整各地形的，从而避免管理不同的vertex buffer、index buffer所带来的复杂度；

## virtural texture

vt相对全面，但不足够详细的介绍[virtual texture](https://zhuanlan.zhihu.com/p/138484024)

### clip map

原版论文为[Clipmap](https://notkyon.moe/vt/Clipmap.pdf)；大致思路为，针对实际的大地图texture，并不加载整个地图texture进入内存；而是，将texture划分为tile，只加载需要的tile进入内存；实现可参考[Clipmaps White Paper](https://developer.download.nvidia.com/SDK/10/direct3d/Source/Clipmaps/doc/Clipmaps.pdf)

需要的tile，实际上需要根据视角，相机位置，屏幕分辨率等因素确定；但是clipmap直接限制了所能使用的tile的数量；直接将mip大于某一数值的tile大小固定，mip小于该数值的使用整个texture tile，从而将整个贴图的mip链，划分为clipmap stack与clipmap pyramid两部分；

实现上，clipmap pyramid使用正常的texture开启mip链来存储，并常驻内存，clipmap stack使用texture array关闭mip来存储，跟进游戏的需要来动态更新里面的内容；

### SVT

svt的一个简易实现可参考[VirtualTexture](https://github.com/jintiao/VirtualTexture)；

Svt直接首先将texture进行划分成page，所有的mip都需要划分为page，所有的page大小都是一致的；

**feedback环节**：运行时，首先跟进屏幕上地形的的uv，与ddx、ddy来确定需要使用哪些page（uv整除来确定page）及对应mip（ddx、ddy来确定mip）；随后将内存中为加载的page进行加载；ddx、ddy的确定需要额外的pass来计算，可以使用低分辨率的rt；这一步，也可以使用基于clipmap的思想，直接根据相机位置，来确定需要哪些page及对应mip，不需要额外pass；feedback的数据甚至可以离线烘焙；

> feedback环节需要从GPU读取屏幕page数据至CPU进行处理，这是一个效率非常低的操作，因为此操作会导致CPU挂机等待GPU任务完成，然后再进行真正的回读操作，会产生比较严重的stall问题，特别针对multi thread rending模式；解决这一问题的方式是使用async回读机制，进行隔帧读取，虽然隔帧读取增加了lantency，但是解决了stall问题，对lantency依赖不深的功能可以使用这种方法；参考[Why is GPU-CPU transfer slow?](https://community.khronos.org/t/why-is-gpu-cpu-transfer-slow/58708)、[AsyncCaptureTest](https://github.com/keijiro/AsyncCaptureTest)

**vt更新环节**：根据加载的内容，以及page的位置及mip，**更新**page table以及physical texture；一般情况下，page table中的一个texel对应了一个page的位置，而page一般采用四四方方的划分方法，因此page table常采用四叉树的形式（即开启mip的texture）来存储physical page的位置（一般存储的信息为page在physical texture中的offset，以及mip）；

假如vt大小为1M x 1M，page大小为256 x 256，那么page table的大小为4k x 4k，page table仍然是一个很大的量；因此可以基于clipmap的思想，将page table使用基于clipmap的形式进行存储，这样page table的大小就进行了很大的优化；前面提到clip map一般使用一个开mip的tex + 一个未开启mip的tex array来存储；也可以进行一些调整，比如移除低mip对应的clipmap pyramid，这样使用一个tex array就够了，甚至可以将tex array展开为2d tex，从而兼容不支持tex array的机型；

**渲染环节**：光照计算与vt关系不大，关系比较大的地方在于贴图的采样；最基本的做法为，跟进当前像素的uv、ddx、ddy来确定page在page table的位置，以及mip，随后根据page table查找真正的page在physical texture上的位置，然后进行采样；

由于physical texture由各个mip下的page拼接而成，那么page的边缘处直接双线性插值就会有问题，因此需要引入border来避免artifact；由于physical texture上各个page的mip是固定的，因此trilinear与anios filter一般就需要手动进行插值计算，或者利用硬件实时更新physical texture的mip，从而支持硬件采样；

### RVT

参考[Terrain in Battlefield 3](https://media.contentapi.ea.com/content/dam/eacom/frostbite/files/gdc12-terrain-in-battlefield3.pdf)以及[Chen_Ka_AdaptiveVirtualTexture](https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2015/presentations/Chen_Ka_AdaptiveVirtualTexture.pdf)
rvt中page的生成是runtime的，从而避免了离线存储大量的page数据；对于需要实时生成的texture具有很大的优势，比如地形所需要的blend后的color、smooth等数值；

RVT与SVT有区别的环节为vt的更新环节，physical texture所需要的page是实时去渲染出来，然后再去更新physical texture；