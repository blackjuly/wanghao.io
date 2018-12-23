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

> 阅读Android 事件分发源码简析（适合阅读人群：有一定Android开发基础，对view 树状视图有一定了解；源码备注：本次分析基于Android 8.0 源码）


## 1.梗概

### 图解事件分发流程

首先，对于刚开始接触阅读Android源码的小伙伴，笔者认为可以借助一些图解流程；先了解一下其背后大致的流程，再扎进源码中进行阅读，避免被太多无关代码干扰无法顺利的阅读主线内容。
那么，既然说到要先用图片大致了解流程，那就先不废话了，直接上图
![Android 事件分发 pipeline](/img/in-post/post-touch-event-dispatch/event-handing-pipeline.JPG) 
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
![范围示意图](/img/in-post/post-touch-event-dispatch/focus-event-handing-part-pipeline.JPG)

### view的分发方法讲解
对于view来讲，我们目前主要关注的方法就是：

>dispatchTouchEvent()

作用：
1. 如果 **View.OnTouchListener.onTouch** 存在发送 event给我们熟知的 **OnTouchListener**
```java
   //部分代码忽略 
  //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
```
2. 如果 **OnTouchListener** 为空，即该event没有被消费，则传递给 **view.onTouchEvent()**,在该method中调用的也就是我们常见
```java
public boolean performClick() {
        // 部分代码忽略
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }
        return result;
    }
```
### viewGroup的分发方法讲解
而对于viewGroup在代码中，主要需要关注的首先同样也是 
**dispatchTouchEvent()**，而viewGroup的dispatch方法与view的作用不同,伪代码如下
```java
 public boolean dispatchTouchEvent ( MotionEvent ev){
     if(!onInterceptTouchEvent){
         for(int i = children.size();i>=0;i--){
             if(children.get(i).dispatchTouchEvent(ev)){
                 return true;
             }
         }
     }
     return super.dispatchTouchEvent(ev);
 }   
```
由伪代码可以看到，在我们可以看到在viewGroup中**dispatchTouchEvent()**的作用：

1. 利用 **onInterceptTouchEvent** 决定是否要截断当前的事件，调用其viewGroup的父类的**dispatchTouchEvent()**

2. 当**onInterceptTouchEvent**为false的时候，viewGroup就分发到其所持有的所有child，倒着依次询问child是否能够消费掉该event

![viewGroup事件分发示意图](http://img.whdreamblog.cn/18-12-21/20913576.jpg)

## 3.view和viewGroup源码详解
开始之前，我们再用一个流程图来简单过一下，代码的一个流程；希望可以帮助小伙伴抓住主线代码进行阅读
![完整分发示意图](http://img.whdreamblog.cn/18-12-21/75470051.jpg)

在看此图的时候，有一点需要注意，我们的view的事件分发的流程一般主要是分析的 Action_Down的情况，而其他状态下的执行路径与我们分析的流程是有不同之处的！

### view源码讲解
首先，我们来看流程的最末端view针对一个event的代码阐述

#### view.dispatchEvent(event)分析

相关讲解链接

[onFilterTouchEventForSecurity](#onFilterTouchEventForSecurity（methd简析）)

[OnTouchListener.onTouch](#OnTouchListener.onTouch（methd简析）)
```java
public boolean dispatchTouchEvent(MotionEvent event) {
        //部分代码忽略
        boolean result = false;
         //部分代码忽略
        final int actionMasked = event.getActionMasked();//1.获取当前event中action
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture 
            //2.为了一些新手势，加一层保护性的清理善后
            stopNestedScroll();//停止了nestedScrollview的滚动
        }

        if (onFilterTouchEventForSecurity(event)) {//3.过滤event中的非安全情况（后文有讲解）
            if ((mViewFlags & ENABLED_MASK) == ENABLED //4.在view状态为enable的情况下
            && handleScrollBarDragging(event)) {//5.如果是鼠标操控滚动栏
                result = true;//直接返回true
            }

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null//6.enable的情况下，且OnTouchListernr不为空
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {//7.直接下发给 onTouch
                result = true;
            }

            if (!result && onTouchEvent(event)) {//8.mOnTouchListener没有消费，则分给其他触控事件；此处注意，result == false才能进入 传递给 onTouchEvent(后文有详细解析)
                result = true;
            }
        }

        // 处理stopNestedScroll和手势的情况，非关注点；暂时忽略
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    }
```
##### onFilterTouchEventForSecurity（methd简析）

该methd是Google为触摸事件的分发指定了一个安全策略：
如果当前View不处于顶部，且View设置的属性是该View不在顶部时不响应触摸事件，则不分发该事件。
即不安全的情况需要满足两点：
1. 在设定被遮挡时需要过滤该事件（mViewFlags包含FILTER_TOUCHES_WHEN_OBSCURED）

2. 当前触控事件确实已经被遮挡（event.getFlags()包含MotionEvent.FLAG_WINDOW_IS_OBSCURED）
```java
  public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // 满足以上两点不安全的情况，直接返回false，下掉该触控事件
            return false;
        }
        return true;
    }
```
##### OnTouchListener.onTouch（methd简析）
我们常用的 onTouch事件，允许用户有机会获取所用event，并且消费掉该事件
```java
    public interface OnTouchListener {
        /**
 *
 * @param v 被触控事件分发到的view
 * @param event The MotionEvent 包含的所有event信息
 * @return True 消费该事件，false不消费  */  
        boolean onTouch(View v, MotionEvent event);
    }
```

#### view.onTouchEvent(event)分析

相关讲解链接

[TouchDelegate.onTouchEvent](#mTouchDelegate.onTouchEvent)

**onTouchEvent**就已经可以看到我们最常用的部分代码内容
```java
 public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();//1.记录坐标点
        final int viewFlags = mViewFlags;
        final int action = event.getAction();//2.获取action

        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE //判断是否开启可以点击的开关
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)//判断长按开关
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;//判断是否可以开启上下文的点击，比如 鼠标右键点击

        if ((viewFlags & ENABLED_MASK) == DISABLED) { //disable的情况处理
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);//设置view内部的Pressed的状态，响应一些比如按下的视图效果
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return clickable; //diable的view也可以消费事件，只是不响应
        }
        if (mTouchDelegate != null) {//如果mTouchDelegate不为空，调用该类（后文讲解) 
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {//可以点击的情况情况进入
            switch (action) {
                case MotionEvent.ACTION_UP:
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    if ((viewFlags & TOOLTIP) == TOOLTIP) {
                        handleTooltipUp();
                    }
                    if (!clickable) {//可以点击时
                        removeTapCallback(); //移除 tap的响应
                        removeLongPressCallback();//移除长按的响应
                        mInContextButtonPress = false;
                        mHasPerformedLongPress = false;
                        mIgnoreNextUpEvent = false;
                        break;
                    }
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                        }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClickInternal();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                        mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                    }
                    mHasPerformedLongPress = false;

                    if (!clickable) {
                        checkForLongClick(0, x, y);
                        break;
                    }

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();

                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true, x, y);
                        checkForLongClick(0, x, y);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    break;

                case MotionEvent.ACTION_MOVE:
                    if (clickable) {
                        drawableHotspotChanged(x, y);
                    }

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        // Remove any future long press/tap checks
                        removeTapCallback();
                        removeLongPressCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```

##### mTouchDelegate.onTouchEvent
此为一个触控的代理类，处理一些特殊场景;最常见的应用场景，就是处理 **拓展view的触控区域**[**Extend a child view's touchable area官方文档链接**](https://developer.android.com/training/gestures/viewgroup#java) ，某一些情况下，我们的view展示必须要比较小，比如一个返回按钮 imageViewButton，但是会造成用户点击很难点击到，所以此时可以继承 TouchDelegate类，处理接收更大区域的点击事件
```java
if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
```