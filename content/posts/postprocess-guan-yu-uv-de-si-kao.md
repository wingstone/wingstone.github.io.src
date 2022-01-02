---
title: '[Thingking] PostProcess关于UV的思考'
date: 2021-06-10 23:47:44
tags: []
published: true
hideInList: false
feature: 
isTop: false
---
以前一直以为屏幕空间下的UV应该是0-1的范围内的；
<!--more-->

## Blit操作下的PostProcess

但是最近在做后处理时，对这件事重新进行了思考，其实Shader中使用的uv肯定是经过光栅化后的数据，既然是光栅化后的数据，那么UV坐标应该是像素中心的UV坐标；

后期一般都是通过DrawMesh来实现的，Mesh里的UV值为0-1，而此时的Mesh肯定需要覆盖整个屏幕，那么可以得到屏幕的最左侧uv.x = 0，最右侧uv.x = 1；而最左侧对应于最左列像素的左边，最右侧对应于最右列像素的右边；

由此可见，对于Vertex Shader，其UV是 **0-1** 范围内的；
对于FragmentSahder，其UV范围是最左列像素坐标到最右侧像素坐标的，令_Size = float4(1/ScreenWidth, 1/ScreenHeight, ScreenWidth, ScreenHeight)，则UV范围为 **(0.5\*_Size.x~1-0.5\*_Size.x, 0.5\*_Size.y~1-0.5\*_Size.y)** ；

下图为10x10分辨率下的UV灰度图：
![](https://wingstone.github.io/post-images/1623341545976.jpg =200x200)
可以看到最左列的灰度值并不为0，看来我们的思考是对的；

## ComputerShader下的PostProcess

ComputerShader（简称CS）下的UV获取比较特殊，因为CS并不走正常的GPU绘制流水线，只是单纯的用多线程进行并行计算，所以我们在CS中获取UV值一般是由ThreadID除以屏幕分辨率来获取；

简易CS Example：
```C
#pragma kernel CSMain

RWTexture2D<float4> _Result;
float4 _Size;   //(1/width, 1/height,  width, height)

[numthreads(8,8,1)]
void CSMain (uint3 id : SV_DispatchThreadID)
{
    float2 uv = id.xy/_Size.zw;
    _Result[id.xy] = uv.x;
}
```

threadID即为我们分配的线程ID，每个线程有唯一值，对于图片处理，一般范围为(0~width-1,0~height-1)，那么算下来，CS里的UV范围应该为 **(0~1-_Size.x, 0~1-_Size.y)** ；

下图为10x10分辨率下的UV灰度图：
![](https://wingstone.github.io/post-images/1623342423558.jpg =200x200)
