---
layout:     post
title:      "Android style & Theme 再探析(二)"
subtitle:   "一统View规范的大杀器——material design"
date:       2018-08-16 20:00:00
author:     "wanghao"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Android
    - 知识记录
    
---

 > 在上一篇android style和Theme的探析中，笔者描述了自己对于它的理解与认识，但是笔者发现仅仅了解这些其实是不够的，在移动应用软件工程化以后，对于像Theme这种全局化的管控view样式来说，十分强大也是意味着十分危险的，因为app在足够庞大以后，Theme的随意改动都会引发，很多页面样式的改变，并且当你不够了解改变会影响到的部分时，这无疑是巨大的风险！


## **View规范的现状**

一般我们谈论的view的规范，一般都提及的是一般资源文件的命名规范，这部分的内容，其实阿里的Android命名规范已经基本覆盖的十分全面了，但是笔者想提及的是一些我们在和UI合作时的一些问题，而这些同样影响着我们的规范......
### **笔者发现的一些问题**
1. 笔者接触到的公司，UI图中文字的尺寸定义一直是一个不确定的事情，我们无法确定各个场景的文字到底应该用哪个尺寸，只能根据自动化标注工具一一仔细定义，无法提取通用
2. 另一个不定的内容，就是我们app中的颜色，是另一大不定因素，目前一般都使用阿里的命名规范，管理颜色

>【推荐】 color 资源使用#AARRGGBB 格式，写入 module_colors.xml 文件中，命
名格式采用以下规则：
模块名_逻辑名称_颜色
如：
```xml
<color name="module_btn_bg_color">#33b5e5e5</color>
```

对于模块的颜色规范我们采用这样的方式，完全没有问题的，并且在后期的国际化中；这样的形式完全也是没有问题的，但是对于一些常用的主题颜色，会一直在很多组件上大量使用，反复重复的颜色；一般这样情况我们都会进行提取，目前见到常用有两种：

1. color命名格式：color_16进制颜色值，如红色 color_ff0000
```xml
<color name="color_FFFFFF">#FFFFFF</color>
```
2. color命名格式：直接用英文，如果颜色多相近则，颜色值后面加数字 red1，如红色 red1
```xml
<color name="red1">#FFFFFF</color>
```

对比两种方式：

**第一种**

优势：替换颜色，添加颜色比较方便，因为直接在标有十六进制的颜色值按规范敲，有没有该颜色即可直接显示
劣势：解释性比较差，直接查看并不能知晓对应的颜色，和直接敲 十六进制颜色值毫无区别

**第二种**

优势：解释性强，更符合我们常见的命名规范
劣势：增加修改不是很方便
例如：
添加三种相近的红色
```xml
   <color name="red1">#xxxxxx</color>
   <color name="red2">#xxxxxx</color>
   <color name="red3">#xxxxxx</color>   
```
由于这三种红色很有可能不是同一时间提出，所以颜色值的数字标号只能以时间顺序作为顺延，而无有浅到深或者由深到浅的顺序，同样不好管理

### **解决方案探究**
虽然这个问题非常细枝末节，但是却困扰了笔者很久，我觉得不可能国外的app开发不遭遇相同的问题，于是我找到了一个开源app  **Materialistic for Hacker News**

styles.xml文件部分代码展示：
```xml
  <!-- =========== -->
    <!-- Text styles -->
    <!-- =========== -->
    <style name="textRankStyle">
        <item name="android:textAppearance">@style/TextAppearance.App.Large</item>
        <item name="android:gravity">center|center_vertical</item>
    </style>

    <style name="textTitleStyle">
        <item name="android:textAppearance">@style/TextAppearance.App.Title</item>
        <item name="android:gravity">center_vertical</item>
        <item name="android:paddingLeft">@dimen/padding</item>
        <item name="android:paddingRight">@dimen/padding</item>
    </style>

    <style name="textSubtitleStyle">
        <item name="android:textAppearance">@style/TextAppearance.App.Subtitle</item>
        <item name="android:singleLine">true</item>
        <item name="android:gravity">center_vertical</item>
        <item name="android:paddingLeft">@dimen/padding</item>
        <item name="android:paddingRight">@dimen/padding</item>
        <item name="android:layout_width">match_parent</item>
        <item name="android:layout_height">wrap_content</item>
    </style>
```
colors.xml文件部分代码展示：
```xml
<color name="red500">#F44336</color>
    <color name="redA100">#FF8A80</color>
    <color name="redA200">#FF5252</color>
    <color name="purpleA400">#D500F9</color>
    <color name="indigoA700">#304FFE</color>
    <color name="lightBlue700">#0288D1</color>
    <color name="lightBlueA700">#0091EA</color>
    <color name="teal50">#E0F2F1</color>
    <color name="teal100">#B2DFDB</color>
    <color name="teal300">#4DB6AC</color>
    <color name="teal400">#26A69A</color>
```
再次浏览过几个开源项目同样有针对文字和颜色的这种写法，再经过几番调查，才真正意识到 原来 **material design**,并不只是给UI设计师看的，而是解决UI和开发的一些沟通问题的一种方案，而不单单只是那一套组件库那么简单

## **Material Design详解**




