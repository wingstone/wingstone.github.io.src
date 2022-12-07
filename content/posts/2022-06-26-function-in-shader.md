---
title: buildin function in shader
author: wingstone
date: 2022-06-26
categories:
- shader
tags:
- shader
metaAlignment: center
coverMeta: out
---

关于内置函数的实现方法，发现[Cg Standard Library](https://developer.download.nvidia.cn/cg/index_stdlib.html)写的最好，给出了各种函数的Reference Implementation；实际上，unity查看shader编译后的代码也可以看到对应实现；这里记录几种常见函数的实现方式，从而可以对shader的复杂度有更好的认知；
<!--more-->

## super function

### sin

采用泰勒级数展开的拟合形式；但原文有说最好实现为native instruction，因此在实际上大概率会直接调用硬件指令来运行；

```c++
float sin(float a)
{
  /* C simulation gives a max absolute error of less than 1.8e-7 */
  float4 c0 = float4( 0.0,            0.5,
                      1.0,            0.0            );
  float4 c1 = float4( 0.25,          -9.0,
                      0.75,           0.159154943091 );
  float4 c2 = float4( 24.9808039603, -24.9808039603,
                     -60.1458091736,  60.1458091736  );
  float4 c3 = float4( 85.4537887573, -85.4537887573,
                     -64.9393539429,  64.9393539429  );
  float4 c4 = float4( 19.7392082214, -19.7392082214,
                     -1.0,            1.0            );

  /* r0.x = sin(a) */
  float3 r0, r1, r2;

  r1.x  = c1.w * a - c1.x;                // only difference from cos!
  r1.y  = frac( r1.x );                   // and extract fraction
  r2.x  = (float) ( r1.y < c1.x );        // range check: 0.0 to 0.25
  r2.yz = (float2) ( r1.yy >= c1.yz );    // range check: 0.75 to 1.0
  r2.y  = dot( r2, c4.zwz );              // range check: 0.25 to 0.75
  r0    = c0.xyz - r1.yyy;                // range centering
  r0    = r0 * r0;
  r1    = c2.xyx * r0 + c2.zwz;           // start power series
  r1    =     r1 * r0 + c3.xyx;
  r1    =     r1 * r0 + c3.zwz;
  r1    =     r1 * r0 + c4.xyx;
  r1    =     r1 * r0 + c4.zwz;
  r0.x  = dot( r1, -r2 );                 // range extract

  return r0.x;
}
```

### asin

asin会转化为泰勒级数展开的拟合形式，在unity查看编译后的源码能发现实现确实如此；

```c++
// Handbook of Mathematical Functions
// M. Abramowitz and I.A. Stegun, Ed.

float asin(float x) {
  float negate = float(x < 0);
  x = abs(x);
  float ret = -0.0187293;
  ret *= x;
  ret += 0.0742610;
  ret *= x;
  ret -= 0.2121144;
  ret *= x;
  ret += 1.5707288;
  ret = 3.14159265358979*0.5 - sqrt(1.0 - x)*ret;
  return ret - 2 * negate * ret;
}
```

### tan

tan会转化为sin与cos来实现；

```c++
float tan(float a) {
  float s, c;
  sincos(a, s, c);
  return s / c;
}
```

```c++
void sincos(float3 a, out float3 s, float3 out c)
{
  s = sin(a);
  c = cos(a);
}
```

### atan

atan会转化为泰勒级数展开的拟合形式；

```c++
float atan(float x) {
    return atan2(x, float(1));
}
```

```c++
float2 atan2(float2 y, float2 x)
{
  float2 t0, t1, t2, t3, t4;

  t3 = abs(x);
  t1 = abs(y);
  t0 = max(t3, t1);
  t1 = min(t3, t1);
  t3 = float(1) / t0;
  t3 = t1 * t3;

  t4 = t3 * t3;
  t0 =         - float(0.013480470);
  t0 = t0 * t4 + float(0.057477314);
  t0 = t0 * t4 - float(0.121239071);
  t0 = t0 * t4 + float(0.195635925);
  t0 = t0 * t4 - float(0.332994597);
  t0 = t0 * t4 + float(0.999995630);
  t3 = t0 * t3;

  t3 = (abs(y) > abs(x)) ? float(1.570796327) - t3 : t3;
  t3 = (x < 0) ?  float(3.141592654) - t3 : t3;
  t3 = (y < 0) ? -t3 : t3;

  return t3;
}
```

### pow

底层会采用exp2与log2来实现，而exp2与log2都会以native的硬件形式来支持，硬件的实现实际上是通过位操作来计算得出（比如对于整数，直接通过位的左移与右移操作来实现）；

```c++
float3 pow(float3 x, float3 y)
{
  return exp2(log2(x)*y);
}
```

### smoothstep

smoothstep依旧以插值的形式来实现；

```c++
float smoothstep(float a, float b, float x)
{
    float t = saturate((x - a)/(b - a));
    return t*t*(3.0 - (2.0*t));
}
```

### reflect

reflect根据反射原理来实现；注意，i向量应朝向平面；

```c++
float3 reflect( float3 i, float3 n )
{
  return i - 2.0 * n * dot(n,i);
}
```

### refract

refract根据折射原理来实现；注意，i向量应朝向平面，eta为extIOR/intIOR；

```c++
float3 refract( float3 i, float3 n, float eta )
{
  float cosi = dot(-i, n);
  float cost2 = 1.0f - eta * eta * (1.0f - cosi*cosi);
  float3 t = eta*i + ((eta*cosi - sqrt(abs(cost2))) * n);
  return t * (float3)(cost2 > 0);
}
```

### normalize

normalize会使用resqrt来实现，而resqrt的实现可能为native instruction，也可能使用pow函数实现；

```c++
float3 normalize(float3 v)
{
  return rsqrt(dot(v,v))*v;
}
```