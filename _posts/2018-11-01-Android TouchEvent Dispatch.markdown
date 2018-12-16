---
layout:     post
title:      "Android touch Event Dispatch"
subtitle:   "Android 事件分发（理论篇）"
date:       2017-07-20 21:36:00
author:     "wanghao"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - 源码
    
---

#Android 事件分发（

> 阅读Android 事件分发源码简析（适合阅读人群：有一定Android开发基础，对view 树状视图有一定了解）


## 1.全面到局部（图解事件分发流程）

首先，对于刚开始接触阅读Android源码的小伙伴，笔者认为可以借助一些图解流程；先了解一下其背后大致的流程，再扎进源码中进行阅读，避免被太多无关代码干扰无法顺利的阅读主线内容。
那么，既然说到要先用图片大致了解流程，那就先不废话了，直接上图
![Android 事件分发 pipeline](/img/in-post/post-touch-event-dispatch/event-handing-pipeline.jpg)
*Android 事件分发 pipeline*

由上图我们可知：
1. 我们的Android系统在分发触控事件时，是先通过一个input Manager Service 将输入的事件传递出来
2. 在分发的最上层的类是window
3. 我们的所有view之间的与其父view的通信都是双向的

所以，依图我们来简单做一个概述，触控事件由 输入service发送给window，由window开始下发，到activity到decoView,到contentView；然后接下来就是我们开发自己的layout部分，层层下发；然后，由最底层开始向上通知一个 result（作用后面详解），告知上层该触控事件的处理情况