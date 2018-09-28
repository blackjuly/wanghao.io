---
layout:     post
title:      "Android style & Theme 再探析(三)"
subtitle:   "自定义Theme，切换皮肤实践"
date:       2018-08-20 19:00:00
author:     "wanghao"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Android
    - 知识记录
    
---

 > 在上一篇android style和Theme的探析中，笔者描述了对于Material Design的理解，本章描述的是笔者在针对之前的md和设计的问题，进行自定义Theme的实践和切换皮肤


## **Theme的设计目的**

使用Material Design的命名规范，自定义Theme和切换皮肤和日夜间模式的知识，进行实践；了解如何对于app主题如何进行覆盖重写；测试各个版本的下的展示情况

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

### **UI调色板 Color Palette**

#### **设计师使用部分**

> 调色板以一些基础色为基准，通过填充光谱来为Android、Web和iOS环境提供一套完整可用的颜色。基础色的饱和度是500。

![material_design ui调色板](http://design.1sters.com/material_design/style/images/style-color-palette-1.png)

这部分虽然是放置在设计模块用于设计师去查看的，但是，实际上我们的对于我们开发使用也有着影响

#### **开发使用展示**
**开发示例**
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
<!-- google's material design colours from 
http://www.google.com/design/spec/style/color.html#color-ui-color-palette -->

    <!--reds-->
    <color name="md_red_50">#FFEBEE</color>
    <color name="md_red_100">#FFCDD2</color>
    <color name="md_red_200">#EF9A9A</color>
    <color name="md_red_300">#E57373</color>
    <color name="md_red_400">#EF5350</color>
    <color name="md_red_500">#F44336</color>
    <color name="md_red_600">#E53935</color>
    <color name="md_red_700">#D32F2F</color>
    <color name="md_red_800">#C62828</color>
    <color name="md_red_900">#B71C1C</color>
    <color name="md_red_A100">#FF8A80</color>
    <color name="md_red_A200">#FF5252</color>
    <color name="md_red_A400">#FF1744</color>
    <color name="md_red_A700">#D50000</color>
    <!--部分代码隐藏-->
</resources>    

```

#### **开发使用分析**
首先，可以看的出我们的开发使用的是笔者在上面提及的第二种颜色值的命名，但是与我们的直接无脑添加数字又不一样的，颜色取值完全根据规范[material 调色板](https://material.io/design/color/the-color-system.html#color-usage-palettes)

**优势**
1. 使得我们颜色数量一定时间内是可控的
2. 它采用饱和度数值做为命名的形式将 **red_100** 和 **#FFCDD2** 关联起来使得UI和开发团队内部讨论颜色全部都有了一个统一名称指代，且以颜色英文开头有解释性
3. 饱和度和颜色深浅具有规律性，使得颜色的排列由浅至深而非以时间排列；且再次插入新的颜色值根据其饱和度和英文名完全可以直接知晓其位置

### **文字样式 typography**

#### **设计师使用部分**

>自从Ice Cream Sandwich发布以来，Roboto都是Android系统的默认字体集。在这个版本中，将Roboto做了进一步全面优化，以适配更多平台。宽度和圆度都轻微提高，从而提升了清晰度，并且看起来更加愉悦。
![字体样式说明](http://design.1sters.com/material_design/style/images/style-typography-roboto-typography.roboto2_specimen_large_mdpi.png)

>**标准样式（Standard Styles）**
<br/>
字体排版的缩放和基本样式（Typographic Scale & Basic Styles）
<br/>
同时使用过多的字体尺寸和样式可以很轻易的毁掉布局。字体排版的缩放是包含了有限个字体
尺寸的集合，并且他们能够良好的适应布局结构。最基本的样式集合就是基于12、14、16、20和34号的字体排版缩放。
<br/>
这些尺寸和样式在经典应用场合中让内容密度和阅读舒适度取得平衡。字体尺寸是通过SP（可缩放像素数，scaleable pixels）指定的，让大尺寸字体获得更好的可接受度。
<br/>
![字体尺寸说明](http://design.1sters.com/material_design/style/images/style-typography-01_large_mdpi.png)

针对Display4，Display3此类字段每一个都是针对一种[场景](https://material.io/design/typography/the-type-system.html#applying-the-type-scale)可以，通过点击链接查看具体场景说明

#### **开发使用展示**
首先，要说明一个概念对于我们以前的文字其实很多方面都是不关注的，加粗斜体透明度字体之类的，而只关注到的只是尺寸；而我们的android 在view的字体使用中，一般其实都不会把字号做为一个单独使用方面,而是用 TextAppearance 这样一个概念
<br/>
**开发示例**
google dimens_material.xml文件
```xml
<resource>
    <dimen name="text_size_display_4_material">71sp</dimen>
    <dimen name="text_size_display_3_material">44sp</dimen>
    <dimen name="text_size_display_2_material">36sp</dimen>
    <dimen name="text_size_display_1_material">27sp</dimen>
    <dimen name="text_size_headline_material">18sp</dimen>
    <dimen name="text_size_title_material">16sp</dimen>
    <dimen name="text_size_subhead_material">16sp</dimen>
    <dimen name="text_size_title_material_toolbar">16dp</dimen>
    <dimen name="text_size_subtitle_material_toolbar">16dp</dimen>
    <dimen name="text_size_menu_material">16sp</dimen>
    <dimen name="text_size_menu_header_material">14sp</dimen>
    <dimen name="text_size_body_2_material">14sp</dimen>
    <dimen name="text_size_body_1_material">14sp</dimen>
    <dimen name="text_size_caption_material">12sp</dimen>
    <dimen name="text_size_button_material">14sp</dimen>
</resource>    
```
goolge style_material.xml
```xml
<resources>
<style name="TextAppearance.Material.Display2">
        <item name="textSize">@dimen/text_size_display_2_material</item>
        <item name="fontFamily">@string/font_family_display_2_material</item>
        <item name="textColor">?attr/textColorSecondary</item>
    </style>

    <style name="TextAppearance.Material.Display1">
        <item name="textSize">@dimen/text_size_display_1_material</item>
        <item name="fontFamily">@string/font_family_display_1_material</item>
        <item name="textColor">?attr/textColorSecondary</item>
    </style>

    <style name="TextAppearance.Material.Headline">
        <item name="textSize">@dimen/text_size_headline_material</item>
        <item name="fontFamily">@string/font_family_headline_material</item>
        <item name="textColor">?attr/textColorPrimary</item>
    </style>
    <!--部分代码省略-->
</resource>        
```

**优势**
1. 所有场景的文字，对于的样式都是一目了然；方便开发归纳总结，提取使用
2. 各个场景的框架已经提出，调整都是针对一种场景进行直接替换；无论是字体还是其他样式都方便管理替换，无需再像我们目前开发状态一般；各个字体设置散落各个页面毫无统一可言

## **总结**
material design在这一番研究后，让我真心觉得其作用不止是提供给设计一种新规范，给开发一套新组件那么简单，更多也是结合了开发app中的实际情况，针对一些内容进行规范，方便我们开发更好的管理应用，当然material design的内容不止这些，笔者只是针对文字和颜色两方面进行了探究；希望更多的特性能被大家发掘出来！