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


## 1.梗概

### 图解事件分发流程

首先，对于刚开始接触阅读Android源码的小伙伴，笔者认为可以借助一些图解流程；先了解一下其背后大致的流程，再扎进源码中进行阅读，避免被太多无关代码干扰无法顺利的阅读主线内容。
那么，既然说到要先用图片大致了解流程，那就先不废话了，直接上图
![Android 事件分发 pipeline](/img/in-post/post-touch-event-dispatch/event-handing-pipeline.jpg)
*Android 事件分发 pipeline*

由上图我们可知：
1. 我们的Android系统在分发触控事件时，是先通过一个input Manager Service 将输入的事件传递出来
2. 在分发的最上层的类是window
3. 我们的所有view之间的与其父view的通信都是双向的

所以，依图我们来简单做一个概述，触控事件由 输入service发送给window，由window开始下发，到activity到decoView,到contentView；然后接下来就是我们开发自己的layout部分，层层下发；然后，由最底层开始向上通知一个 result（作用后面详解），告知上层该触控事件的处理情况。

### decoView部分知识补充说明

Android界面与类对应图

这里补充一点我们android源码view部分的组织结构；首先在我们的activity中，我们都知道一个非常常见的方法

>setContentView(R.layout.xxx);

相信很多朋友都不陌生，而这个方法的作用就在于，我们的Activity是持有一个decoView的view对象；而setContentView正是将我们开发者的布局设置到这个decoView中进行呈现；而这个decoView的作用也很明显，由图可以看到，我们的导航栏和状态栏都会被decoView与开发者的布局拼凑到一起形成一个界面。

## 2.view与viewGroup执行流程

### 讲解范围概述
在上一部分讲解，相信大家对于我们的事件传递大致有了一个概念，接下来我们来用伪代码表述和流程图初步对代码有个认识（PS：笔者不主张一开始就直接讲解源码细节，容易让人一头雾水，最好可以有材料对相关知识点有个大致了解，有一个主线，避免在源码中迷失），那么，我们首先要划定一个范围，我们目前只针对 activity以下，更多重心会放在事件在开发者的布局传递的一个流程！

讲解范围示意图

### view的分发方法讲解
对于view来讲，我们目前

### viewGroup的分发方法讲解

### view和viewGroup交互流程示意图

## 3.view和viewGroup源码详解
