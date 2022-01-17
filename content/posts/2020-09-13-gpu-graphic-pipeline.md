---
title: 'Pipeline——GPU Graphic Pipeline（图形管线）'
date: 2020-09-13 09:14:06
tags: [pipeline,图形学]
published: true
hideInList: false
feature: 
isTop: false
---
## 管线介绍

所谓管线就是一个流程，针对硬件来说，处理一个图元有一个硬件渲染流程**Graphic Pipeline（图形管线）**；针对实际应用来说，渲染一帧画面也需要一个渲染流程**Render Pipeline/Path（渲染管线/路径）**；Graphic Pipeline处于更加低级的渲染层次，是渲染一个物体必走的渲染流程；
<!--more-->

## GPU Graphic Pipeline

具体的管线流程要看实际的硬件驱动，Direct3D每一个版本都有很大的改动，这里以Direct3D11为例进行介绍；具体的文章参考可以看这里：[Direct3D 11 Graphics Pipeline](https://docs.microsoft.com/en-us/windows/win32/direct3d11/overviews-direct3d-11-graphics-pipeline)，[Direct3D 12 Graphics Pipeline](https://docs.microsoft.com/en-us/windows/win32/direct3d12/pipelines-and-shaders-with-directx-12)；

### Input-Assembler Stage（图元装配阶段）

这一阶段主要进行图元的装配，先从用户填充的缓冲中读取数据，然后将数据装配成图元；此阶段可装配成不同的图元类型（如 line lists, triangle strips, or primitives with adjacency）

### Vertex Shader Stage（顶点着色阶段）

**一个可编程shader阶段**，此阶段主要处理IA阶段输入的顶点，执行每顶点的处理（如变换、蒙皮、变形，顶点光照等）；VS阶段总是处理单一顶点，并输出单一顶点；VS阶段必须处于激活状态，VS必须提供；

### Tessellation Stages（细分阶段）

该阶段实际上有三个小阶段来完成图元的细分；通过硬件实现细分，GPU Graphic Pipeline能将低细节的模型转换为高细节模型进行渲染；

#### Hull-Shader Stage（壳着色阶段）

**一个可编程shader阶段**，用来生成一个patch（和patch constants），每个patch对应一个输入的patch（quad, triangle, or line）；有点像一个基本的图元类型；

#### Tessellator Stage

一个固定处理阶段，用来生成简单格式的域，一个域代表一个geometry patch并用来生成更小物体的集合（triangles, points, or lines），通过连接domain sample来实现；

#### Domain-Shader Stage（域着色阶段）

**一个可编程shader阶段**，用来计算每个domain sample的顶点的位置，

### Geometry Shader Stage（几何着色阶段）

**一个可编程shader阶段**，该阶段同样以顶点作为输入，以顶点作为输出；但与VS有很大不同；

1. 输入顶点数不一定为一，输入顶点数刚好可以可组成一完整图元（two vertices for lines, three vertices for triangles, or single vertex for point）；并且可以携带邻接的图元顶点数据（an additional two vertices for a line, an additional three for a triangle）；
2. 输出顶点数不一定为一，输出的顶点数目可以形成特定的拓扑结构即可，输出的拓扑结构可选（GS stage output topologies available are: tristrip, linestrip, and pointlist）；

### Stream-Output Stage（流输出阶段）

该阶段的目的在于能够从不断的GS阶段输出顶点数据，至一个或多个缓存中；

### Rasterizer Stage（光栅化阶段）

此阶段，将每个图元光栅化为像素，通过顶点差值来计算像素信息；光栅化过程总是运行顶点裁剪，透视除法，将顶点转换为齐次裁剪空间，然后将顶点映射到视口上；

### Pixel Shader Stage（像素着色器阶段）

**一个可编程shader阶段**，该阶段能够使用更加丰富的着色技术（每像素光照，后处理效果）；该阶段能够将常量，纹理数据，顶点差值数据以及其他数据进行计算，从而产生像素的输出；

### Output-Merger Stage（输出合并阶段）

该阶段生成最终的像素颜色，通过管线状态的整合，即：PS阶段生成的数据，如render target、depth/stencil buffer；OM阶段是用来决定可见颜色的最后一步，并且会Blend最终的像素颜色；
