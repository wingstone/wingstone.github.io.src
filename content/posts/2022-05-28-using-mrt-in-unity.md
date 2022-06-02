---
title: using mrt in unity
author: wingstone
date: 2022-05-28
categories:
- multi render texture
- unity
tags:
- 心得
metaAlignment: center
coverMeta: out
---

记录一下在unity中使用mrt的一些方法及问题；
<!--more-->

## 使用Graphics.SetRenderTarget

此方法可以用于后期中的mrt使用，需要先设置mrt，然后再blit到null；也可以搭配Graphics.RenderMesh或Graphics.DrawMesh通过绘制quad来实现后期中的mrt使用；如果把quad修改成逐物体的mesh，则可以实现逐物体的mrt渲染；

也可以在`OnPreRender`中进行调用，这样可以直接在相机的绘制中，针对整个场景使用mrt；`OnPreRender`由于是每帧调用的，因此在此函数中`Graphics.SetRenderTarget`也是每帧都进行设置；

如果针对场景使用mrt，会产生与下面的Camera.SetTargetBuffers方法一样问题，解决方法参考下面；

比较适用于buildin管线。

使用方法为：

```c#
var _mrt = new RenderBuffer[2];
var rt1 = RenderTexture.GetTemporary(source.width, source.height, 0, RenderTextureFormat.Default);
var rt2 = RenderTexture.GetTemporary(source.width, source.height, 0, RenderTextureFormat.DefaultHDR);

_mrt[0] = rt1.colorBuffer;
_mrt[1] = rt2.colorBuffer;

// Blit with a MRT.
Graphics.SetRenderTarget(_mrt, rt1.depthBuffer);
Graphics.Blit(null, _material, 0);

// Combine them and output to the destination.
_material.SetTexture("_SecondTex", rt1);
_material.SetTexture("_ThirdTex", rt2);
Graphics.Blit(source, destination, _material, 1);

RenderTexture.ReleaseTemporary(rt1);
RenderTexture.ReleaseTemporary(rt2);
```

## 使用Camera.SetTargetBuffers

此方法只能在相机的绘制中，针对整个场景使用mrt，用法类似于Camera.targetTexture，设置一次即可；

比较适用于buildin管线。

使用方法为：

```c#
void Start()
{
    cam = GetComponent<Camera> ();
    cam.hdr = true;

    rts = new RenderTexture[bufferNames.Length];
    buffers = new RenderBuffer[bufferNames.Length];
    for (int i = 0; i < rts.Length; i++) {
        rts [i] = new RenderTexture ((int)cam.pixelWidth, (int)cam.pixelHeight, 0, RenderTextureFormat.ARGBFloat);
        rts [i].filterMode = FilterMode.Point;
        rts [i].name = bufferNames [i];
        rts [i].Create ();
        buffers [i] = rts [i].colorBuffer;
    }
    dRt = new RenderTexture ((int)cam.pixelWidth, (int)cam.pixelHeight, 24, RenderTextureFormat.Depth);
    dRt.name = depthBufferName;

    cam.SetTargetBuffers (buffers, dRt.depthBuffer);
}
```

需要注意的是，使用了MRT后，很多其他的后期插件（比如官方的postprocess stack）都不兼容，主要问题是使用mrt导致官方后其中的BuiltinRenderTextureType.CameraTarget丢失，就算自己去写后期也有一定的兼容问题；

比较好的做法是手动创建一个备用相机来渲染mrt，然后将备用相机合成结果blit到当前相机所用的CameraTarget上；

或者手动创建一个RT作为finalRT，在相机渲染结束后，将mrt的结果合成到finalRT上，再将finalRT blit到屏幕上（通过绘制api或通过rawimage）；

在unity的论坛上，官方提供了一个方法来解决这个问题，就是在OnPostRender中，将mrt的合成结果blit到屏幕上；方法为

```C#
void OnPostRender()
{
    Display screen1 = Display.displays[0];
    Graphics.SetRenderTarget(screen1.colorBuffer, screen1.depthBuffer);

    m_mrtCombine.SetTexture( "_target2", m_target2 );
    Graphics.Blit( m_target1, null, m_mrtCombine );
}
```

## 使用CommandBuffer.SetRenderTarget

使用此方法拥有最大的自由度，可用于在相机的绘制中使用mrt，也可以在后期中使用mrt；此方法也是unity所推荐的方法，即可用于srp，也可用于buildin管线；

事实上，unity的整个srp都是由commandbuffer所构建的，那么mrt的支持自然不会落下；

使用方法为：

```c#

// Define command buffer
var cmd = new CommandBuffer();
cmd.name = “TestGBufferCMD”;

// Define render identifier and set textures
var rtGBuffersID = new RenderTargetIdentifier[rtGBuffers.Length];
for (var i = 0; i < rtGBuffers.Length; ++i) {
    rtGBuffersID[i] = cmd.GetTemporary(Screen.width, Screen.height, 0);
}
depthBuffer = cmd.GetTemporary(Screen.width, Screen.height, 0);

// Set up command buffer
cmd.SetRenderTarget(rtGBuffersID, depthBuffer);
cmd.ClearRenderTarget(true, true, Color.clear, 1);

// draw render in srp
cmd.DrawRenderer(targetRender, gBufferMaterial);

// draw big triangle for postprocess
cmd.DrawMesh(fullscreenTriangle, Matrix4x4.identity, propertySheet.material, 0, pass, propertySheet.properties);

// Add command buffer to camera 
Camera.main.AddCommandBuffer(CameraEvent.BeforeForwardOpaque, cmd);

// Or execute command buffer manually
Graphics.ExecuteCommandBuffer(cmd);
```


## Reference

1. [Multiple render target (MRT) questions for Unity Technologies](https://forum.unity.com/threads/multiple-render-target-mrt-questions-for-unity-technologies.262966/)