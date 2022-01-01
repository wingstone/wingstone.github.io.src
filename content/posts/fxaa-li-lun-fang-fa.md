---
title: 'FXAA理论方法'
date: 2021-03-01 22:52:01
tags: []
published: true
hideInList: false
feature: 
isTop: false
---
FXAA是基于图像空间理论的抗拒齿方法；

## 基础理论为：

1. 进行图像边缘检测；
2. 针对检测出来的图像边缘，进行抗拒齿处理；

## 基础步骤：

1. **采样屏幕颜色，转换至亮度；**
Nvidia建议使用red以及green通道来计算亮度进行优化，人眼对这两种颜色最为敏感；亮度计算方法如下：
```c++
float FxaaLuma(float3 rgb) {return rgb.y * (0.587/0.299) + rgb.x; }
```

2. **根据亮度来计算对比度，用对比度来作为边缘检测的标准；**
一般边缘检测都会使用**当前像素**，以及**上下左右四个像素**来进行对比度的计算；当对比度大于某个阈值时，认为当前点处于边缘位置；
```c++
float3 rgbN  = FxaaTextureOffset(tex, pos.xy, FxaaInt2( 0,-1)).xyz;
float3 rgbW  = FxaaTextureOffset(tex, pos.xy, FxaaInt2(-1, 0)).xyz;
float3 rgbM  = FxaaTextureOffset(tex, pos.xy, FxaaInt2( 0, 0)).xyz;
float3 rgbE  = FxaaTextureOffset(tex, pos.xy, FxaaInt2( 1, 0)).xyz;
float3 rgbS  = FxaaTextureOffset(tex, pos.xy, FxaaInt2( 0, 1)).xyz;
float lumaN  = FxaaLuma(rgbN);
float lumaW  = FxaaLuma(rgbW);
float lumaM  = FxaaLuma(rgbM);
float lumaE  = FxaaLuma(rgbE);
float lumaS  = FxaaLuma(rgbS);
float rangeMin = min(lumaM, min(min(lumaN, lumaW), min(lumaS, lumaE)));
float rangeMax = max(lumaM, max(max(lumaN, lumaW), max(lumaS, lumaE)));
float range = rangeMax -rangeMin;
if(range < max(FXAA_EDGE_THRESHOLD_MIN, rangeMax * XAA_EDGE_THRESHOLD))           {return FxaaFilterReturn(rgbM); }
```

3. **边缘的横纵向检测，用于后续抗拒齿处理；**
边缘进行光栅化之后，在比较小的维度上，只有横向边缘与纵向边缘之分；检测出来横向与纵向，可以便于后续沿边缘进行延伸；
```c++
float edgeVert = abs((0.25 * lumaNW) + (-0.5 * lumaN) + (0.25 * lumaNE)) +abs((0.50 * lumaW ) + (-1.0 * lumaM) + (0.50 * lumaE )) +abs((0.25 * lumaSW) + (-0.5 * lumaS) + (0.25 * lumaSE));
float edgeHorz = abs((0.25 * lumaNW) + (-0.5 * lumaW) + (0.25 * lumaSW)) +abs((0.50 * lumaN ) + (-1.0 * lumaM) + (0.50 * lumaS )) +abs((0.25 * lumaNE) + (-0.5 * lumaE) + (0.25 * lumaSE));
bool horzSpan = edgeHorz >= edgeVert;   //判断当前像素位于横向边缘还是纵向边缘
```

4.** 判断真正的边缘边界位置，并将当前点定位到真正的边缘边界处；**
检测出横向还是纵向后，需要定位边缘边界是在上横向还是下横向，左横向还是右横向；
当前点是在像素中心，定位出边缘边界（位于两个像素的交接处）后，即可将当前点移动到边缘边界；
```c++
// 梯度最大处作为边缘边界的位置；
float luma1 = isHorizontal ? lumaDown : lumaLeft;
float luma2 = isHorizontal ? lumaUp : lumaRight;
float gradient1 = luma1 - lumaCenter;
float gradient2 = luma2 - lumaCenter;
bool is1Steepest = abs(gradient1) >= abs(gradient2);

// 我也不清楚此处为什么要用0.25来进行normalize，感觉就是一个简单的factor；
float gradientScaled = 0.25*max(abs(gradient1),abs(gradient2));

float stepLength = isHorizontal ? inverseScreenSize.y : inverseScreenSize.x;

//边缘边界处（计算像素相交处）的亮度平均值
float lumaLocalAverage = 0.0;
if(is1Steepest){
    stepLength = - stepLength;
    lumaLocalAverage = 0.5*(luma1 + lumaCenter);
} else {
    lumaLocalAverage = 0.5*(luma2 + lumaCenter);
}

// 将uv移动半个像素，定位到边缘边界处（像素相交处）.
vec2 currentUv = In.uv;
if(isHorizontal){
    currentUv.y += stepLength * 0.5;
} else {
    currentUv.x += stepLength * 0.5;
}
```

5. 定位到边界处，需要沿边缘方向（横向或纵向）进行搜索，来计算当前边缘的长度；
之所以计算边缘的长度，是因为锯齿一般都出现在边缘端点处；离端点越远，越不容易产生锯齿；我们需要计算当前边缘边界距边缘端点的距离，用来计算锯齿的强弱程度；

```c++
// 第一步迭代
vec2 offset = isHorizontal ? vec2(inverseScreenSize.x,0.0) : vec2(0.0,inverseScreenSize.y);
vec2 uv1 = currentUv - offset;
vec2 uv2 = currentUv + offset;

float lumaEnd1 = rgb2luma(texture(screenTexture,uv1).rgb);
float lumaEnd2 = rgb2luma(texture(screenTexture,uv2).rgb);
lumaEnd1 -= lumaLocalAverage;
lumaEnd2 -= lumaLocalAverage;

bool reached1 = abs(lumaEnd1) >= gradientScaled;
bool reached2 = abs(lumaEnd2) >= gradientScaled;
bool reachedBoth = reached1 && reached2;

if(!reached1){
    uv1 -= offset;
}
if(!reached2){
    uv2 += offset;
}  

// 继续后面的迭代
if(!reachedBoth){

    for(int i = 2; i < ITERATIONS; i++){
        if(!reached1){
            lumaEnd1 = rgb2luma(texture(screenTexture, uv1).rgb);
            lumaEnd1 = lumaEnd1 - lumaLocalAverage;
        }
        if(!reached2){
            lumaEnd2 = rgb2luma(texture(screenTexture, uv2).rgb);
            lumaEnd2 = lumaEnd2 - lumaLocalAverage;
        }
        reached1 = abs(lumaEnd1) >= gradientScaled;
        reached2 = abs(lumaEnd2) >= gradientScaled;
        reachedBoth = reached1 && reached2;

        // 此处QUALITY为迭代步进的缩放系数，一般前5步为1，后面可逐渐递增
        if(!reached1){
            uv1 -= offset * QUALITY(i);
        }
        if(!reached2){
            uv2 += offset * QUALITY(i);
        }

        if(reachedBoth){ break;}
    }
}
```

6. 偏移（此偏移量作用于最初uv，用于最终采样）估算
前面说到越靠近边缘终点，锯齿越严重，那么最终像素输出采样点的偏移量越大；与此相反，越靠近边缘中点，输出采样点的偏移量越小；算法如下：
```c++
// 使用边缘边界点与边缘终点的距离处于边缘的长度来计算offset
float distance1 = isHorizontal ? (In.uv.x - uv1.x) : (In.uv.y - uv1.y);
float distance2 = isHorizontal ? (uv2.x - In.uv.x) : (uv2.y - In.uv.y);
bool isDirection1 = distance1 < distance2;
float distanceFinal = min(distance1, distance2);
float edgeThickness = (distance1 + distance2);
float pixelOffset = - distanceFinal / edgeThickness + 0.5;

// 添加额外的判断，来保证端点处的走势（gradient）与中心处的一致
bool isLumaCenterSmaller = lumaCenter < lumaLocalAverage;
bool correctVariation = ((isDirection1 ? lumaEnd1 : lumaEnd2) < 0.0) != isLumaCenterSmaller;
float finalOffset = correctVariation ? pixelOffset : 0.0;
```

7. 子像素抗拒齿处理
使用额外的步骤即可处理子像素抗拒齿（即细线引起的锯齿，或点引起的锯齿，我也不知道为啥叫做子像素锯齿==）；
基本思路是计算中心点与周围点的差值，然后计算此差值与局部差值的比值来计算子像素抗拒齿的偏移量；最终选择较大偏移量作为最终采样值；
```c++
float lumaAverage = (1.0/12.0) * (2.0 * (lumaDownUp + lumaLeftRight) + lumaLeftCorners + lumaRightCorners);
float subPixelOffset1 = clamp(abs(lumaAverage - lumaCenter)/lumaRange,0.0,1.0);
float subPixelOffset2 = (-2.0 * subPixelOffset1 + 3.0) * subPixelOffset1 * subPixelOffset1;
float subPixelOffsetFinal = subPixelOffset2 * subPixelOffset2 * SUBPIXEL_QUALITY;

// Pick the biggest of the two offsets.
finalOffset = max(finalOffset,subPixelOffsetFinal);
```

8. 使用最终偏移量进行采样;
```c++
vec2 finalUv = In.uv;
if(isHorizontal){
    finalUv.y += finalOffset * stepLength;
} else {
    finalUv.x += finalOffset * stepLength;
}

vec3 finalColor = texture(screenTexture,finalUv).rgb;
fragColor = finalColor;
```

注意点：FXAA运行在sRGB空间；直接作用于linear、hdr数据，会出现闪烁问题；