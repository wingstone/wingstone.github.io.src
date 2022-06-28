---
title: why triple frame buffer
author: wingstone
date: 2022-06-28
categories:
- game engine
tags:
- game engine
metaAlignment: center
coverMeta: out
draft: true
---

介绍为什么需要triple framebuffer，推荐[What is V-Sync? Should You Turn it on or off?](https://www.hardwaretimes.com/what-is-v-sync-should-you-turn-it-on-or-off/)
<!--more-->

先说结论，single buffer会产生tearing问题；double buffer配合v-sysc可以解决这个问题，但在running rate小于refresh rate时会带来performence问题（比如refresh rate为60，running rate为45，最终的target rate为30）；引入triple buffer可以解决double buffer带来的performence问题；
