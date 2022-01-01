---
title: 'NPR之描边'
date: 2020-09-28 15:49:37
tags: [图形学,trick]
published: true
hideInList: false
feature: 
isTop: false
---
**对于非平滑模型，采用背部扩展描边的方式，容易出现断裂问题；**

解决方法为：**使用高模的平滑法线来进行背面的扩展**，可以使用法线贴图计算高模法线；

对于NPR渲染，其实一般也不怎么使用贴图；所以可以**将高模的法线bake到顶点色**里面；

由于蒙皮网格会实时计算变换后的切空间下的normal、tangent、binormal；因此不能bake local space下的高模法线，除非将local space下的高模法线bake到tangent或binormal进行存储，或者将tangent space下的高模法线bake到顶点色里进行存储；