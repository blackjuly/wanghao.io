---
layout:     post
title:      "Android style & Theme 再探析(四)"
subtitle:   "业务实践心得总结"
date:       2018-08-17 21:20:00
author:     "wanghao"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Android
    - 知识记录
    
---

> 基于前几篇的探究，我们已经有了对一个app的theme和style的理论探究和demo描述了不少，那么接下来针对我在自己在app生产环境中的使用的一个使用心得总结


## **Android Theme的生产环境使用总结**

<br/>
本篇主要针对小伙伴如果要在自己的app中实践，整理以前混乱的theme和style的一个实践
<br/>

### **针对Theme配置的一些建议**

如果大家按本文第一章的方式设定Theme时，一定要注意几点：

 文字颜色一定要设置小心，由于Theme具有继承性，所以文字颜色的设定会被Android本身的style的继承结构沿用到其子类；但是，在沿用其子类时，有时会进行一定配置的重写，而此时重写的配置就会变为一个不可控因素，产生意想不到的bug

示例：
笔者在使用时，设定了这样的配置

```xml
<style name="ThemeBase" parent="Theme.AppCompat.Light.NoActionBar">
        <!--忽略部分代码-->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="buttonStyle">@style/EhiTheme.Button</item>
    </style>
    <!--主题修改为使用最新-->
    <style name="EhiTheme" parent="ThemeBase">
        <!--定义v21 之前api的内容-->
    </style>
      <!--重写button默认样式-->
     <style name="EhiTheme.Button" parent="@style/Widget.AppCompat.Button">
        <item name="android:background">@drawable/btn_default_ehi_selector</item>
        <item name="android:textColor">@color/white</item>
    </style>

```
代码可以看的出，此为一个button的默认样式修改；而这份代码关键就在于

```xml
<item name="android:textColor">@color/white</item>
```

这句代码将按钮文字设置为白色十分危险，因为Android中很多控件都会沿用这部分的样式；但是，同时这个部分的样式针对各个版本的Android系统上也是有着不同的体现！

以下是不同版本下，此重写样式后AlertDialog的不同体现

Android 8.0

Android 6.0

通过上图可以看到，在Android 6.0的弹窗上，文字采用了colorAccent的颜色，而Android8.0则采用了文字继承下来的白色；但是两者在显示上的一个共同点就是，背景都被置空；这就造成了白色背景加上白色字体的显示bug！

a>同时,要特别注意的一点是，这样的bug不止存在alertDialog，同样会反映在 timePickDialog和datePickDialog上面！
b>另外，需要大家注意的是，在v7包下的alertDialog和app包下的alertDialog是读取的两套设定

v7包：
```xml
 <item name="alertDialogTheme"></item>
```
app系统包下：
```xml
<item name="android:alertDialogTheme"></item>

```
如果项目在使用上不规范的情况下，很可能两种dialog都会引入进行使用，那么两种其一不起作用，都会有产生Bug的风险！
解决方案：
针对Dialog的Theme主题部分，进行重写

定义Alert自己的Theme：

colors.xml
```xml
<color name="colorAccent">#FFFF7E00</color>
```
Themes.xml
```xml
    <style name="EhiTheme.AlertDialog" parent="Theme.AppCompat.Light.Dialog.Alert">
        <item name="colorAccent">@color/colorAccent</item>
        <item name="android:textColor">@color/colorAccent</item>
        <item name="buttonStyle">@style/EhiTheme.Button.Alert</item>
    </style>

    <style name="EhiTheme.Button.Alert" parent="@style/Widget.AppCompat.Button">
        <item name="android:background">@color/white</item>
        <item name="android:textColor">@color/colorAccent</item>
    </style>

<style name="ThemeBase" parent="Theme.AppCompat.Light.NoActionBar">
        <!--定义v7 之后所有api的内容-->
        <!--基本主题色-->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <!--部分忽略-->
        <item name="buttonStyle">@style/EhiTheme.Button</item>
        <item name="alertDialogTheme">@style/EhiTheme.AlertDialog</item>
         <item name="android:alertDialogTheme">@style/EhiTheme.AlertDialog</item>
    </style>
       
```
v21中Themes.xml
```xml
     <style name="EhiTheme" parent="ThemeBase">
        <!--定义v21 之后api的内容-->
        <item name="android:timePickerDialogTheme">?alertDialogTheme</item>
        <item name="android:datePickerDialogTheme">?alertDialogTheme</item>
        <item name="android:alertDialogTheme">?alertDialogTheme</item>
    </style>
```

代码中，如果结构不好有多个baseActivity基类一定要注意！父类在继承**FragmentActivity**和**AppCompatActivity**时，其展现形式是不一样的，由于AppCompatActivity在为了保证变为统一样式在内部做了很多封装；（ps:theme继承了的情况下**Theme.AppCompat.Light.NoActionBar**；使用基础activity会crash），需要在添加后测试一下

### **针对view自定义属性**

自定义控件的属性的命名的一点小建议:

以前我们的自定义view属性的命名全凭喜好
主流的一般有这样的，小驼峰的命名形式
```xml
<declare-styleable name="EhiTitleBar">
    
        <attr name="isSearchView" format="boolean"/>
        <attr name="searchViewHint" format="string" />
        <attr name="titleBackground" format="color|reference" />
    </declare-styleable>
```
还有下划线大法的
```xml
<declare-styleable name="EhiDrawingBoard">
        <attr name="stroke_width" format="integer"/>
        <attr name="paint_color" format="color"/>
        <attr name="canvas_color" format="color|reference"/>
        <attr name="anti_alias" format="boolean"/>
    </declare-styleable>
```

但是我认为很多内容的使用，我们都应该更接近原生控件的使用：

例如textView
```xml
<TextView
        android:id="@+id/text_type"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:text="今天"
        android:layout_marginLeft="10dp"
        android:textColor="@color/black"
        android:textSize="18sp" />

```
可以看的出android的控件命名其实是分成两部分
属性是作用于其子类本身的内部，像这样的文本尺寸
>   android:textSize="18sp" 

采用了小驼峰法

而属性比如是作用于控件的类似layoutparam这种和父类相关的属性
>  android:layout_height="wrap_content"

以下划线来区分，这样其实我觉得对于学习和接受度其实都可以很快，毕竟在使用原始控件时的方式都类似

### **Style命名的一点小建议**

style的使用命名规范的一点小建议
 
 以前的style我们是这么命名的：
 ```xml
 <!-- 条目标签样式 -->
    <style name="item_reimburse_label">
        <item name="android:layout_width">0dp</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:layout_weight">0.3</item>
    </style>
 ```
但是，这种命名其实非常凌乱，
+ 看不出style的层级关系，继承关系
+ 看不出它所属的模块；所以在比较大的项目工程中，在上方使用者调用会非常混乱
+ style的统一管理十分困难，有差不多的一组style有样式改动，将会是一个灾难
+ 组件化后，各个模块容易出现命名重复问题

后来我们进行了一点优化

```xml
<!--myorder是module名-->
 <style name="myorder_item_reimburse_label">
        <item name="android:layout_width">0dp</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:layout_weight">0.3</item>
    </style>
```

命名虽然看的出模块所属，但是我觉得没有充分将style的功能发挥出来，使用也并不怎么友好！

我认为比较适合我们在大型项目中工程化的style的命名是需要更向系统的style命名形式靠拢的：

一般我们可以采用大驼峰的命名形式，以 . 作为各种应用场景的区分 

 **Module.页面.控件类型.控件修饰描述**


```xml
    <!--总模块-->
    <style name="CompanyInfo" />
     <!--总模块，所属界面-->
    <style name="CompanyInfo.DriverMangerSearchOrderResult" />
     <!--总模块，所属界面.对应控件-->
    <style name="CompanyInfo.DriverMangerSearchOrderResult.ItemLabelContent">
        <item name="android:layout_width">wrap_content</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:layout_marginTop">@dimen/tv_order_content_margin_top</item>
        <item name="android:ellipsize">end</item>
    </style>
      <!--总模块，所属界面.对应控件.控件应用场景描述-->
     <style name="CompanyInfo.DriverMangerSearchOrderResult.ItemLabelContent.NoBackground">
        <item name="android:background">@null</item>
    </style>
```
可是，我们很多情况下会有很多界面共用同样的样式，所以为了这种情况，我们可以利用 style的显示继承
首先，定义一个公共模块的样式
```xml
<!--labelContentView的Style-->
    <style name="EhiBase.Widget.LabelContent">
        <!--部分忽略-->
        <item name="contentColor">@color/colorGray100</item>
        <item name="contentHintColor">@color/colorGray350</item>
    </style>
```
然后，在自己的module里面直接进行引用,这样，在同时保证了命名的统一的同时，还兼顾的使用通用样式
```xml
<!--用于个人管理模块-->
    <style name="PersonalManager" />
    <!--个人信息界面-->
    <style name="PersonalManager.MyInformation" />
    <!--利用显示继承，使用通用样式-->
    <style name="PersonalManager.MyInformation.LabelContent" parent="EhiBase.Widget.LabelContent"/>
     
```
这样有以下优势
+ 有统一的模块描述，页面描述，控件描述；
+ 能够更好使得命名空间不冲突
+ 同时利用 style的显式继承可以做到对控件的统一管控
+ 类似Android原生的用法，上手更快

部分源码展示
```xml
 <style name="Theme.AppCompat" parent="Base.Theme.AppCompat"/>
    <style name="Theme.AppCompat.CompactMenu" parent="Base.Theme.AppCompat.CompactMenu"/>
    <style name="Theme.AppCompat.DayNight" parent="Theme.AppCompat.Light"/>
    <style name="Theme.AppCompat.DayNight.DarkActionBar" parent="Theme.AppCompat.Light.DarkActionBar"/>
    <style name="Theme.AppCompat.DayNight.Dialog" parent="Theme.AppCompat.Light.Dialog"/>
    <style name="Theme.AppCompat.DayNight.Dialog.Alert" parent="Theme.AppCompat.Light.Dialog.Alert"/>
```
隐式继承控制命名的完整性，显式继承控制模块与模块之间的通用部分



