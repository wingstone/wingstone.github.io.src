---
title: 'why sdf is better'
author: wingstone
date: 2022-09-12
categories:
- contents
metaAlignment: center
coverMeta: out
---

SDF简短心得

<!--more-->

为什么会有sdf字体比普通的字体质量高很多，哪怕是在同样的贴图分辨率下；

根本原因在于sdf存储的信息量比普通的字体贴图要大得多；

sdf存储的是整个场的信息，每个像素的信息都是有意义的，用来重建整个距离场；特别是在双线性采样（整体非线性，各个维度线性）的加持下，可以使得整个场的重建并不需要很高的分辨率（高频信息除外）；

而普通字体容易出现依赖分辨率的问题；原因在于普通字体贴图的信息之存在于边界上，边界内外的纯白与纯黑色都属于浪费掉的贴图信息；

更近一步，普通字体相当于使用一个bit的存储位来存储的距离场；自然不如使用多个bit位存储的sdf质量高；

至于SDF所带来的Blend优势，属于距离场定义所天然带来的优势，可以更好的处理sdf blend所带来的边界变化；