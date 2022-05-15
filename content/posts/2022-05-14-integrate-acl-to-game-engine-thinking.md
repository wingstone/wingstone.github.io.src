---
title: Integrate acl to game engine thinking
author: wingstone
date: 2022-05-14
categories:
- animation compression
- game engine
tags:
- 心得
metaAlignment: center
coverMeta: out
---

记录一下阅读acl unreal plugin，以及将acl集成到游戏引擎中的一些使用心得；
<!--more-->

## acl的使用方法

acl的插件使用方法，在仓库中有非常完整的文档说明，并且有配套的博客来讲解背后的实现原理；

### acl的验证

在acl的tools文件夹包含一些简单的压缩与解压example，比较适合单独验证acl的压缩与解压缩；

对于没用过acl的人，可以将自己的单个动画片段的数据整理后，整合acl做一个小程序进行压缩与解压缩的验证；写法可参考tools文件夹下的example；

从我验证的结果来看，acl压缩的质量非常之高，并且使用非常低的内存与磁盘存储，并且动画效果肉眼几乎看不出压缩的痕迹；

### acl的引擎集成

如果完成了acl的验证过程，基本上对acl的集成有了充分的认识，那么集成acl就相对简单点；

比较好的集成方式，是参考[acl-ue4-plugin](https://github.com/nfrechette/acl-ue4-plugin)的实现，整个结构非常清晰与完整，对引擎不会有太大改动；

整个集成过程可分为三步：

压缩过程：

1. 准备骨骼动画数据，将引擎内部的骨骼动画数据转换为acl需要的acl::track_array_qvvf类型；
2. 设置压缩参数，比如压缩误差、压缩质量等；
3. 调用acl压缩接口，将压缩结果存至以内存块，并将内存块的二进制数据以引擎内部存储形式存储至文件；

解压过程：

1. 加载压缩数据，cast为acl使用acl::compressed_tracks类型；
2. 进行必要的解压缩初始化流程；
3. 在动画数据采样时，调用acl提供的decompress_track流程即可；

## acl使用一些注意事项

acl需要的骨骼动画数据，需要为原生的骨骼动画数据，并且骨骼动画数据需要保存动画数据之间的骨骼链信息，并且需要每个track的samplerate、length、keycount保持一致；准备数据时务必要保证这一步满足要求；

acl只提供了uniform sampling的算法，并没有提供keyframe reduction的算法，因此想要集成keyframe reduction算法的话，需要自己重新去写一套，工作量应该会比较大；

存储与加载acl压缩数据时，还需要注意另外一个细节就是字节对齐，acl::compressed_tracks是16字节对齐的；

## 关于acl的一些感想

acl厉害的一点是采用了更加合理地误差计算（考虑了误差沿骨骼链传递的因素），并且针对uniform sampling，采取了更合理的误差优化算法；

如果想要集成Keyframe Reduction算法，acl所使用的误差计算方式是非常值得借鉴的，而误差优化算法就需要去精心设计（也就是抽取哪一帧更加合理）；更进一步，Keyframe Reduction过后的数据，仍可以使用量化的算法来进行存储（仅仅是数据的量化）；这是一个很大的工程量；

acl关于骨骼动画压缩着一块，基本上只能适用于骨骼动画；但是acl还提供了标量数据的压缩，标量数据的压缩显得更加的通用，只要提供合适的误差算法，使用acl进行标量数据的压缩也是一步很大的优化；事实上，acl-ue4-plugin里所提供的关于blendshape动画的压缩，就是采用acl标量数据的压缩方法；

## Reference

1. [acl](https://github.com/nfrechette/acl)
2. [acl-ue4-plugin](https://github.com/nfrechette/acl-ue4-plugin)
3. [Animation Compression: Table of Contents](https://nfrechette.github.io/2016/10/21/anim_compression_toc/)