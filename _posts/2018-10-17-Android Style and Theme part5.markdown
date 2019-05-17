---
layout:     post
title:      "Android style & Theme 再探析(五)"
subtitle:   "浅谈 Android Theme & Style 优秀的XML架构设计 ——Theme继承链讲解（补充篇）"
date:       2018-08-17 21:20:00
author:     "wanghao"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Android
    - 知识记录
    
---

> 基于前几篇的探究，我们已经有了对一个app的theme和style的理论探究和实战都有了些心得，一直以来在Android领域，我们对于各种 MVVM，MVC的架构代码的一个基础架构有着一个广泛的讨论，但是相对于火热的JAVA代码的架构，在Android中的Theme & Style 的结构却鲜有人提起，其实一个很简单的原因是大家鲜有深度去使用Style和Theme；并且国内的UI设计并没有什么公认的规范，这对于theme和style的使用，又增加了一块绊脚石；不过，了解一下Android theme和style方面的设计，更有助于我们对于软件工程管理，JAVA 代码非常清晰，资源代码凌乱不已这都是对于我们的工程有巨大危害！

## Theme的业务背景

### 提一个很多Android工程中常见的场景
为了方便理解，我们从使用者的角度；从包装的最外层去分析；一般我们在一个工程很常见的部分就是，由android studio生成的

```xml
  <style name="ThemeBase" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
         <item name="colorAccent">@color/colorAccent</item>
    </style>
```
类似这样的一份Theme主题文件。而此时，如果我们突然需要在当前theme中添加一个属性

```
<item name="android:windowTransluscentStatus">true</item>
```
而此时比如你的工程当前 **minSdk = 16** 而这个属性要求的**minSdk = 19**的情况下，你会获得
```
 xxxx(eg:android:windowTransluscentStatus) requires API level 19 (current min is 16)
```
这样的一个错误警告！为了解决这个错误，我们一般会做以下的处理

* 将当前默认版本中的 高版本api（eg:android:windowTransluscentStatus）移除
* 新建对应版本的value文件夹(eg:values-v19) 
* 新建一个同样的themes.xml 将对应的api放入其中

此时在v19的文件夹下面

```xml
  <style name="ThemeBase" parent="Theme.AppCompat.Light.NoActionBar">
      <item name="android:windowTransluscentStatus">true</item>
    </style>
```
加入新版的属性后，我们的其他属性也必须让v19以上的用户同时享受到

那么我们可以有以下几种解决方案

### 愚蠢的解决方案
全部复制一遍,弊端很明显；每添加一个属性都得同时复制多遍，维护成本巨大

### 常规的解决方案

#### 方案简介
定义一个themeBase在基础的values中的themes.xml 管理基础的api属性

```xml
 <style name="ThemeBase" parent="Theme.AppCompat.Light.NoActionBar">
        <!--定义v7 之后所有api的内容-->
        <!--基本主题色-->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <!-- Window 背景色/控件没有设置背景时, 默认的颜色-->
        <item name="android:windowBackground">@color/colorGray500</item>
        <!--背景色-->
        <item name="android:colorBackground">@color/colorGray500</item>
    </style>
    
    <!--主题修改为使用最新-->
    <style name="MyTheme" parent="ThemeBase">
        <!--定义v19 之前api的内容-->

    </style>  
```
利用继承定义一个 **MyTheme**继承自基础属性，定义低版本可以用的api；在**values-v19** 的theme.xml 定义同样的内容
```xml
 <style name="MyTheme" parent="ThemeBase">
        <!--定义v19 之后api的内容-->
        <item name ="android:windowTransluscentStatus">true</item>
    </style>
```

总的来说这一种方式解决了一定的问题，至少一段时间内我们的属性都不会产生任何的问题；但是，对于向后兼容性来说还是存在一样的问题！

#### 当更多新特性来袭（方案弊端）
比如，当我们在v21时，需要添加新的属性

在values-v21的themes.xml中
```xml
<resources>
    <style name="MyTheme" parent="ThemeBase">
        <item name="android:windowSharedElementEnterTransition">@android:animator/fade_in</item>
    </style>
</resources>
```
而此时，我们再次运行我们的app，**android:windowTransluscentStatus**便不再生效了，会造成这样的问题；很明显我们是继承了同样的父类导致的，转化成代码可能更清晰一点

value 中
```java
class MyTheme extents ThemeBase{
    
}
```
value-v19 中
```java
class MyTheme extents ThemeBase{
    private boolean windowTransluscentStatus = true;
}
```
value-v21 中
```java
class MyTheme extents ThemeBase{
    private int windowSharedElementEnterTransition = R.animator.fade_in;
}
```

想要能够value-v21实现同样的效果，我们看起来需要这样做
```xml
<resources>
    <style name="MyTheme" parent="ThemeBase">
        <item name="android:windowSharedElementEnterTransition">@android:animator/fade_in</item>
        <item name ="android:windowTransluscentStatus">true</item>
    </style>
</resources>
```
而一旦你这样做了，便是灾难的开始！因为，在后面越来越多的版本中添加新的属性，都要将老的属性拷贝一遍；无疑这是相当不可取的！所以，接下来让我们来看看来自google官方更优的解决方案

## 更优的方案3：Android theme 继承链
继续刚才那个问题，刚才说到我们在使用方案的困境就在于当遭遇超过两个版本的新版api需要添加时，单一父类，多个子类的树状结构就不再适用了，而其实如果你由看过android的theme源码，这部分其实我们可以这样解决：

在values的themes.xml中
```xml
    <style name="Base.V7.AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Generic, non-specific attributes -->
         <!--定义v7 之后所有api的内容-->
        <!--基本主题色-->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <!-- Window 背景色/控件没有设置背景时, 默认的颜色-->
        <item name="android:windowBackground">@color/colorGray500</item>
        <!--背景色-->
        <item name="android:colorBackground">@color/colorGray500</item>
    </style>
    
    <style name="MyTheme" parent="Base.V7.AppTheme"/>
```
在values-v19的themes.xml中，继承 **Base.V7.AppTheme**并且放入该api指定属性
```xml
    <style name="Base.V19.AppTheme" parent="Base.V7.AppTheme">
         <!--定义v19 api的内容-->
         <item name="android:windowTranslucentStatus">true</item>
    </style>
    <style name="MyTheme" parent="Base.V19.AppTheme"/>
```
在values-v21的themes.xml中，继承 **Base.V19.AppTheme**并且放入该api指定属性
```xml
    <style name="Base.V21.AppTheme" parent="Base.V19.AppTheme">
         <!-- API 21 specific attributes -->
        <item name="android:windowSharedElementEnterTransition">@android:animator/fade_in</item>
    </style>
      <style name="MyTheme" parent="Base.V21.AppTheme"/>
```
最终各个版本的theme，指定一个相同名字的子类，**MyTheme**,在AndroidManifest.xml中指定下来，即可
```xml
 <application
        android:theme="@style/Mytheme"
        tools:ignore="GoogleAppIndexingWarning"
        tools:replace="android:label"
        >
```

我们关注几个跟前面方案的不同点：

继承的父类不再是以 扁平树状结构，而是改以一个继承链的形式出现
```
<style name="Base.V7.AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
   <style name="Base.V19.AppTheme" parent="Base.V7.AppTheme">
   <style name="Base.V21.AppTheme" parent="Base.V19.AppTheme">

```
通过强大的xml继承结构做到了完美的属性向后兼容且版本管理清晰，维护可靠每个对应版本的api只需要定义一遍，
是不是似曾相识，有没有很像我们在 appcompat中android官方的管理方式？

```xml
<!--摘选自 android values-->
 <style name="Base.Theme.AppCompat" parent="Base.V7.Theme.AppCompat">
<!--摘选自 android values-v21-->
  <style name="Base.V21.Theme.AppCompat" parent="Base.V7.Theme.AppCompat">
 <style name="Base.Theme.AppCompat.Light" parent="Base.V21.Theme.AppCompat.Light"/>
```

没错，这种方案的来源就是来自官方的 appCompat的思路！同时，同样的思路我们也可以用于我们的style；而使用了这样的方式，无疑中可以让我们适配更多种类的设备 无论是 value-某api又或者是， values-sw600dp类似针对宽屏等等的设计，都将更加舒适！良好的命名规范搭配具有强大拓展性的xml让我们的android在UI的适配和兼容性真的做到独树一帜！

## 优势总结
* 维护方便，每个版本的特性只需要定义一遍，且命名规范清晰查看方便；与系统定义方式相同故也方便其他人接受
* 拓展方便在不同的系统api版本下，以及未来的新版本中都可以方便定义新的api属性！
* 方便我们的自定义控件在不同版本加入不同新特性拓展性！