---
title: 'digital human'
author: wingstone
date: 2022-07-23
categories:
- contents
metaAlignment: center
coverMeta: out
---

数字人相关技术记录

<!--more-->

## 眼球

### 眼球瞳孔折射

#### 使用视差贴图来计算折射后的贴图采样位置（unreal）

实现方法：首先在tangent空间对view进行折射计算，然后使用raymatching的方式来计算折射后view与高度图相交的位置，使用高度图相交点的uv作为折射后uv采样点；

优点为：使用范围广，对模型及贴图没有过多的要求；
缺点为：视差贴图需要进行大量的高度图采样，需要考虑其性能问题（不过眼球一般所占屏幕范围都比较小，还是可以考虑使用的）；
 
#### 基于模型空间瞳孔的高度来计算折射位置（unity）

实现方法：首先在object空间，对view进行折射计算，随后在object空间根据瞳孔的高度，计算折射后view在瞳孔平面偏移的距离，使用偏移后的点在瞳孔平面的投影来作为uv采样点；

优点为：不需要进行多采样即可完成瞳孔的折射操作，使用眼球在瞳孔平面的投影作为采样的uv，便于进行程序化眼球生成；不需要高度图，使用模型来控制瞳孔的高度；
缺点为：对模型有一点要求，模型的原点必须为瞳孔中心且模型朝向需与shader一致（便于根据高度计算折射偏移量），模型不能进行蒙皮（蒙皮会改变模型本身的原点，从而无法得到正确的瞳孔高度，也无法在瞳孔平面进行正确的投影计算uv），如果有动画需求，可将眼球挂接在骨骼下面，控制骨骼的动画；

## reference

1. [Jorge Jimenez](http://www.iryoku.com/)
2. [com.unity.demoteam.digital-human.sample](https://github.com/Unity-Technologies/com.unity.demoteam.digital-human.sample)
3. [Digital Humans](https://docs.unrealengine.com/4.27/en-US/Resources/Showcases/DigitalHumans/)