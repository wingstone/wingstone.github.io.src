---
title: '贴图技术——Parallax Mapping（视差贴图）'
date: 2020-09-30 13:10:15
tags: [图形学]
published: true
hideInList: false
feature: 
isTop: false
---
视差贴图属于位移贴图(Displacement Mapping)技术的一种，它对根据储存在纹理中的几何信息对顶点进行位移或偏移。一般使用位移贴图之前，需要对模型进行细分（细分着色器），然后进行顶点位移；

位移贴图要想有好的效果，需要大量顶点支持，而使用视差贴图即可省去大量的顶点使用；

视差贴图的原理实际上是对采样纹理坐标进行偏移，而偏移的原理根据视角观察高度图的真实过程进行模拟，因此可以模拟出真实的贴图凹凸遮挡关系；

在实际的使用过程中，一般使用 **深度图来代替高度图** ，两种互为反相；

## Parallax mapping

最原始视差贴图方法就叫Parallax mapping，其大概原理是：在切线空间下，当前采样坐标获得的高度作为V向量偏移的长度，然后偏移后的长度在切平面的投影即为坐标的偏移量

```C++
//_ParallaxHeight为一控制参数
uv -= tex2D(_HightMap, uv)*TV.xy*_ParallaxHeight;     //V向量为切空间下的向量
```
在高度图变化比较剧烈的地方，采用这种方法会有很对问题；因此又发展出了Steep Parallax Mapping（陡峭视差贴图）；

## Steep Parallax Mapping

对于高度变化剧烈地方，其实很难通过一步就定位到偏移后的位置，陡峭视差贴图方法实际上就是ray matching方法，使用此方法可以更精确的定位到偏移后位置；但由于ray matching算法的性质，在计算量不足的情况下，容易出现分层的痕迹；针对分层问题，后面提出了Parallax Occlusion Mapping(视差遮蔽映射)方法；

该算法首先将深度值进行分层，每一步递进一层；若当前步所获取深度值与步进深度值的差值的符号，与前一步相反，则可定位当前层深度即为偏移深度；分层数量越多，计算量越大，计算结果越精细；

```C++
float numLayers = 10;
float depthStep = 1.0 / numLayers;
float currentDepth = 0.0;
float2 uvStep = TV.xy * _ParallaxHeight/ numLayers; 
float currentDepthMapValue = tex2D(_DepthTex, uv).r;

for(int i = 0; i< 10; i++)
{
    if(currentDepth > currentDepthMapValue)
        break;
    uv -= uvStep;
    currentDepthMapValue = tex2D(_DepthTex, uv).r;
    currentDepth += depthStep;  
}
```
## Parallax Occlusion Mapping

视差遮蔽贴图与陡峭视差贴图类似，区别仅在于最后深度的选择；陡峭视差贴图最终的深度为步进深度的倍数，并且最终深度大于此位置的采样深度，而前一步的深度小于前一步位置的采样深度；针对此现象，我们可以在两者之间进行深度插值，这样最终采样深度就是一个连续的分布了；

插值的根本原理就是相似三角形原理；代码如下，示意图请看参考引用；

```C++
float numLayers = 10;
float depthStep = 1.0 / numLayers;
float currentDepth = 0.0;
float2 uvStep = TV.xy * _ParallaxHeight/ numLayers; 
float currentDepthMapValue = tex2D(_DepthTex, uv).r;

float preDiff = 1e-5;       //防止除0
for(int i = 0; i< 10; i++)
{
    if(currentDepth >= currentDepthMapValue)
    {
        float curDiff = currentDepth-currentDepthMapValue;
        uv -= uvStep*preDiff/(curDiff+preDiff);
        break;
    }
    else
        preDiff = currentDepthMapValue - currentDepth;
    uv -= uvStep;
    currentDepthMapValue = tex2D(_DepthTex, uv).r;
    currentDepth += depthStep; 
}
```

## Relief Mapping

视差遮蔽贴图在最后的一步采用插值进行计算，本质上是采用线性分布模拟步进深度中的非线性深度分布；因此最好的方法就是采用更细的步进深度进行测试，但这样就要求需要有大量的步进运算；因此可采用二分法方法来进行步进，这样可以更快定位到相交的深度；但是二分法要求单调性，实际的深度分布不可能是但单调分布的，因此最佳的步进方法为，先进行大步进深度的传统线性步进，再进行二分步进，这就是Relief Mapping的中心思想；

```C++
//linear rematching
float numLayers = 10;
float depthStep = 1.0 / numLayers;
float currentDepth = 0.0;
float2 uvStep = TV.xy * _ParallaxHeight/ numLayers; 
float currentDepthMapValue = tex2D(_DepthTex, uv).r;

for(int i = 0; i< 10; i++)
{
    if(currentDepth > currentDepthMapValue)
    {
        break;
    }
    uv -= uvStep;
    currentDepthMapValue = tex2D(_DepthTex, uv).r;
    currentDepth += depthStep;  
}

//binary rematching
for(int i = 0; i< 8; i++)
{
    uvStep *= 0.5;
    depthStep *= 0.5;
    currentDepthMapValue = tex2D(_DepthTex, uv).r;
    if(currentDepth < currentDepthMapValue)
    {
        uv -= uvStep;
        currentDepth += depthStep;
    }
    else
    {
        uv += uvStep;
        currentDepth -= depthStep;
    }
}
```

实际上，深度图的使用原理，与地形及其类似；确定好视线与深度相交点后，就可以沿光源方向进行阴影计算，[PDF：Relief Mapping](https://shintaroiguchidotcom.files.wordpress.com/2016/01/relief-mapping-in-a-pixel-shader-using-binary-search.pdf)这篇文章也提到了使用深度图进行自投影的计算；

关于使用raymatching方法进行地形的渲染，可以参考[iq：raymarching terrains](https://www.iquilezles.org/www/articles/terrainmarching/terrainmarching.htm)，读完后相应对于视差贴图的理解可以更深一步；

## 个人思考

从结果上来看，确实算法越来越精确了，但是都要有一个严重的问题：如下图所示：
![实际运行图片](https://wingstone.github.io/post-images/1601477127814.jpg)
可以看出，在视差起作用的边界处，出现了明显的锯齿，感觉如何解决这个要命的问题才是当下需要着重研究的；

## 参考

[learnopengl：Parallax-Mapping](https://learnopengl.com/Advanced-Lighting/Parallax-Mapping)
[learnopengl中文版：Parallax-Mapping](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/05%20Parallax%20Mapping/)
[PDF：Relief Mapping](https://shintaroiguchidotcom.files.wordpress.com/2016/01/relief-mapping-in-a-pixel-shader-using-binary-search.pdf)
[Wiki：Parallax mapping](https://en.wikipedia.org/wiki/Parallax_mapping)