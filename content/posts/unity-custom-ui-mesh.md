---
title: 'Unity Custom UI Mesh'
date: 2020-05-11 14:34:08
tags: [unity]
published: true
hideInList: false
feature: 
isTop: false
---

介绍在unity中实现自定义ui的形状；

## Unity Custom UI Mesh

unity中继承Graphic类或者MaskableGraphic可以实现自定义UI组件，并且在`protected override void OnPopulateMesh(UI.VertexHelper vh);`函数中可以操作或自定义网格，用于实现各种各样的UI形状；

这篇[文章](https://www.hallgrimgames.com/blog/2018/11/25/custom-unity-ui-meshes)介绍了在unity中实现自定义ui的过程；

我在我的个人项目中[UnityScriptTools](https://github.com/wingstone/UnityScriptTools)添加了圆形、矩形、圆角矩形的ui实现；

在实际过程中，可能大部分情况是使用透明的贴图来实现相应形状的ui，但是一些情况使用自定义ui形状更合适，有些情况甚至必须使用自定义ui才能做；