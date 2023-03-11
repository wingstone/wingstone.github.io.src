---
title: 'Rendering in sky children of light'
date: 2022-11-14
categories:
- Capture Frame
tags: 
- Capture Frame
- Pipeline
draft: true
---

光遇渲染效果截帧分析，抓帧使用的一加手机pro7，屏幕分辨率为3120x1440 ；这里做一个粗略的渲染分析，细节上需要慢慢研究；
<!--more-->

## Copy/Clear Pass

一些Copy buffer to image：
一个2d array的天空盒iamge

![](skybox.png)

两张light image，用于tiled lighting以及tiled shadow；光遇里面阴影没有使用shadowmap之类的，使用的应该是capsule shadow之类的；

![](wind0.png)
![](wind1.png)

一张未知的image

![](unknown0.png)

一张用来最后渲染至屏幕时，控制镜头畸变的程度(中间拉伸，两边挤压)；此图的rg通道是有微弱渐变的；
![](unknown1.png)


## Pass 1 (2 Targets)

rt1用于存储角色移动轨迹，用于沙地的渲染，
rt2用于存储角色移动产生的浅波法线轨迹，用于草地与水面的渲染；
两张rt的格式都为R16G16B16A16_FLOAT，分辨率都为256x256；

![](trail.png)

## Pass 2 (1 Targets + Depth)

rt分辨率为1200x552，格式为R11G11B10_FLOAT与D32；

用于不透场景以及角色的渲染，DrawIndexed指令与DrawIndexedIndirect混用，没有看到instancing的使用，像蜡烛这些应该可以使用instancing来进行优化；

首先使用一个indirect draw完成predepth的绘制，绘制内容为场景大块物体；

![](predepth.png)

随后开始正常绘制，首先是角色及部分其他物体的绘制；

![](character.png)

然后是地表场景的绘制；

![](scene.png)

### 地表场景的特殊处理

1. 不同材质以不同pass不同mesh来绘制
2. 材质以blend的形式去绘制，在材质重叠处进行混合处理
3. 利用前面的predepth来保证没有错误blend效果
4. weight mask存储在mesh的顶点色中

石头材质的绘制

![](stone.png)

贴图不多，主要是石头法线贴图的tiling

![](stonenormal.png)

沙材质的绘制

![](sand.png)

沙的模型

![](sandmesh.png)

模型里所自带的weight mask

![](weight.png)

草的绘制

![](grass.png)

地表场景基本没有手绘颜色贴图，都是一些细节noise贴图，保证了整体细节的精细度与可控性；沙的做法可以直接参考journey的分享；

## Pass 3 (1 Targets)

rt分辨率为300x138，格式为R32_FLOAT；

此pass使用上一个pass得到的深度图，来得到低分辨的深度图；

![](lowdepth.png)

## Pass 4 (1 Targets)

rt分辨率为150x69，格式为R32_FLOAT；

此pass使用上一个pass得到的低分辨率场景深度图，以及一些云的3d sdf，来生成低分辨写云模型的深度图；

此时的云模型深度图并不精确，更加靠近相机，为了提高raymatching的效率，而进行的更低分辨率的pre rematching；

![](clouddepth0.png)

使用的输入图：

![](cloud0.png)
![](cloud1.png)
![](cloud2.png)
![](cloud3.png)

## Pass 5 (1 Targets)

rt分辨率为300x128，格式为R32_FLOAT；

此pass使用上上一个pass得到的低分辨率场景深度图，以及上一个pass的低分辨率云层深度，还有一些云的3d sdf，来生成当前分辨率云模型的深度图；

当前分辨率的深度图用于后面云渲染时使用；

![](clouddepth1.png)

## Pass 6 (1 Targets + Depth)

rt分辨率为600x276，格式为R16G16B6A16_FLOAT与D32；

此pass进行真正的云彩渲染；

输入贴图为：

角色轨迹图

![](trail.png)

云的建模及noise图

![](cloud0.png)
![](cloud4.png)
![](cloud5.png)
![](cloud6.png)
![](cloud7.png)
![](cloud1.png)
![](cloud2.png)
![](cloud3.png)

场景深度图以及云深度图

![](lowdepth.png)
![](clouddepth1.png)

最终的输出为：

![](cloudexport.png)

## Pass 7 (2 Targets + Depth)

rt分辨率为1200x552，格式为R11G11B10_FLOAT、R16G16B6A16_FLOAT与D32；

此pass主要进行半透物体的渲染，如草，烛火，飞行轨迹等；

在渲染前，首先使用云的深度，地表场景的深度合成整个场景的深度，并写入depth；

### 草的绘制

草的绘制使用的是vkCmdDraw指令，但是拓扑使用的是point list；

![](grassvert0.png)

草的颜色存储在顶点色里面，有明显视锥剔除的痕迹；

![](grassvert1.png)

有点像untiy地形系统或者粒子体统的的做法，但是这个草的量也太大了；


----

**下面为低分辨率下，遇境的抓帧分析；**

## Color pass（1 target + depth）

云跟水以低分辨率渲染到同一个rt上，同时会写入对应的深度信息；
目标贴图为512*236的color与depth，格式为R16G16B16A16_FLOAT与D32

### 云的渲染

以全屏的方式渲染，使用的rt有3张：
一张如下：
![](image1.png)
一张低分辨率深度图
![](image2.png)
一张低分辨率云深度图
![](image3.png)
其他贴图分别如下，基本上都是3d贴图，有noise贴图，有3d filed贴图：
![](image4.png)
![](image5.png)
![](image6.png)
![](image7.png)
![](image8.png)
![](image9.png)

![](image10.png)

![](image11.png)

### 水的渲染

Mesh：27360个顶点，以视锥体对齐的方式来渲染，有两个插槽，一个是position另一个不确定；
![](image12.png)
![](image13.png)
![](image14.png)
贴图：
纯黑透明rt，暂不确定用途（应该是用于动态的浅波方程计算）；
![](image15.png)
带有dither信息的cubemap贴图，用于远处水面反射；
![](image16.png)
两张用于forward+计算的光照信息图；
![](image17.png)
![](image18.png)
法线贴图；
![](image19.png)
屏幕颜色rt，用于近处水面反射，使用raymatching获取；
![](image20.png)
两张不同分辨率与格式的深度rt；
![](image21.png)
![](image22.png)

## Color pass（2 targets + depth）
目标贴图为1024*472的color与depth，格式为R11G11B10_FLOAT、R16G16B16A16_FLOAT与D32;

这个pass渲染都是半透的问题，第二个rt基本上都是作为mask来使用的，如下图所示，不知为何格式这么浪费；

![](image31.png)

### 全屏深度合成（big triangle）：
将前面云、水的深度与场景不透物体的深度进行合成写入：
输入rt：
![](image23.png)
![](image24.png)
输出rt：
![](image25.png)
### 草的渲染
mesh输入：
拓扑直接使用point list，以point的形式来显示；避免使用更多的mesh；
![](image26.png)
草的culling肯定是做了的view culling很明显
![](image27.png)
总共使用三张图
交互的波动图，vs使用
![](image28.png)
风吹动的噪音图，vs使用
![](image29.png)
草的形状贴图
![](image30.png)

## Color pass（1 target） 

### 体积光
目标贴图为512*236，格式为R16_FLOAT，无depth；用于渲染体积光;

mesh为

![](image32.png)
![](image33.png)

输入贴图为

![](image34.png)
![](image35.png)

输出贴图为

![](image36.png)

## Color pass（1 target）

### 合成最终图像

目标贴图为1024*472的color，格式为R16G16B16A16_FLOAT;
输出贴图为下图，应该为了更好的利用精度，将颜色映射到了0-1范围；合成时还会考虑自爆光的影响；

![](image37.png)

输入图为：

第一张图用来渲染天空时使用；

![](image38.png)
![](image39.png)
![](image40.png)
![](image41.png)
![](image42.png)
![](image43.png)
![](image44.png)
![](image45.png)

## Color pass（1 target + depth）

### 深度拷贝并计算motion vector

目标贴图为512*236的color与depth，格式为R16G16B16A16_FLOAT与D32;

此时合成的静态场景的深度以及云、水的深度；

贴图输入为：

第一张贴图应该为静态场景的深度信息；
![](image46.png)
![](image47.png)
![](image48.png)

输出贴图b通道为主，有微弱的rg通道，推测rg为motion vector，b为depth；

![](image49.png)

### 动态物体深度及motion vector

绘制白鸟以及角色的深度信息；角色的vs需要考虑贴图扰动的影响；

## Color pass（1 target）

### 深度预处理

目标贴图为512*236的color，格式为R16G16_FLOAT;

此时合成的静态场景的深度以及云、水的深度；

贴图输入为上一pass的输出；

贴图输出为深度预处理后的motion vector，用于taa，可参考inside的做法；如下图：

![](image51.png)

## Color pass（1 target）

### taa

目标贴图为1024*472的color，格式为R16G16B16A16_FLOAT;

与inside做法基本一致，输入贴图为：motion vector + 当前渲染图 + 上一帧taa输出；

![](image52.png)

输出为正常的taa后结果；

## Color pass（1 target）

### 空间切换

目标贴图为512*236的color，格式为R16G16B16A16_FLOAT;

将taa后的贴图转换为正常颜色空间；输出贴图为下图；输出时还会在a通道存储动态物体的深度信息（云、人、白鸟），深度有取临近像素最近距离的处理；

![](image53.png)


## 一系列Color pass（1 target）

目标贴图格式为R11G11B10_FLOAT，用于pingpong pass 进行bloom；

## 最终Color pass（1 target）

### 贴图合成

目标贴图为3120*1440的color，格式为R8G8B8A8_SRGB;

输入贴图为：前面的bloom贴图、taa后贴图，空间转换后贴图，还有一未知贴图，如下图所示：

![](image54.png)

这里分辨率有一较大的分歧，需要仔细查看如何处理的；

### ui绘制

随后以全分辨率绘制ui即可；

## Color pass（1 target）

目标贴图为2x2的color，格式为R8G8B8A8_UNORM;

### 自爆光处理

自爆光需要统计像素的亮度信息；这里统计的是空间转换后的贴图；
输入贴图为：

![](image55.png)

输出贴图为：

![](image56.png)