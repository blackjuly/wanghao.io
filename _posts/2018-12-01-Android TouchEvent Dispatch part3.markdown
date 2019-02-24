---
layout:     post
title:      "Android touch Event Dispatch part3"
subtitle:   "Android 事件分发（实战篇—— 处理ScollView和listView的滑动冲突）"
date:       2018-11-20 21:30:00
author:     "wanghao"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - 源码
    
---

# Android 事件分发—滑动冲突解决
## 0.背景知识补充
>在Android的触摸消息中，已经实现了三种监测，它们分别是
>1. pre-pressed：对应的语义是用户轻触(tap)了屏幕
>2. pressed：对应的语义是用户点击(press)了屏幕
>3. long pressed：对应的语义是用户长按(long press)了屏幕

下图是触摸消息随时间变化的时间轴示意图：

其中，t0和t1定义在ViewConfiguration类中，标识了tap和longpress的超时时间，定义如下：

```java
/**
     * Defines the duration in milliseconds we will wait to see if a touch event 
     * is a tap or a scroll. If the user does not move within this interval, it is
     * considered to be a tap. 
     */
    private static final int TAP_TIMEOUT = 115; // t0
    
    /**
     * Defines the duration in milliseconds before a press turns into
     * a long press
     */
    private static final int LONG_PRESS_TIMEOUT = 500; // t1

```
## 1.梗概

上一节我们详细介绍了，ACTION_DOWN的分发流程，本节来简单讲解下ACTION_MOVE与ACTION_UP的一个大致流程分析，因为这两者的代码流程基本一致，所以一并进行一个讲解

## 3.view和viewGroup源码详解
开始之前，我们再用上一节的一个流程图来简单回顾一下ACTION_DOWN的流程
![完整分发示意图](http://img.whdreamblog.cn/18-12-21/75470051.jpg)

而ACTION_UP和ACTION_DOWN不同的一点就在于，当图中最后的VIEW在接收到ACTION_DOWN的通知，对该触控事件不感兴趣时；则之后的ACTION就不再向下分发！

### view源码讲解
在view的源代码中，dispatchTouchEvent()方法部分ACTION_UP和ACTION_DOWN部分执行的主流程部分并没有什么区别，主要区别在于
view.onTouchEvent部分，需要进行一定讲解

#### view.onTouchEvent(event)分析

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

      //部分源代码忽略
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {//a.可以点击的情况情况；b.有悬停和长按显示工具提示框
            switch (action) {
               
                  case MotionEvent.ACTION_UP:
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    if ((viewFlags & TOOLTIP) == TOOLTIP) {
                        handleTooltipUp();//暂时忽略
                    }
                    if (!clickable) {//不可以点击时，然后跳出switch
                        removeTapCallback(); //移除 tap（敲打屏幕）的响应
                        removeLongPressCallback();//移除长按的响应
                        mInContextButtonPress = false;
                        mHasPerformedLongPress = false;
                        mIgnoreNextUpEvent = false;
                        break;
                    }
                    //获取prepressed状态
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;

                    //如果是pressed状态或者是prepressed状态，才进行处理
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {//可获取焦点，可获取焦点的触控模式，且没有获取焦点的情况下，获取到焦点
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);//显示按住的状态
                        }
                       
                        // 是否处理过长按操作了，如果是，则直接返回
            	        // 进入该代码段，说明这是一个tap操作，首先移除长按回调操作
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            //只有显示了 pressed状态，才执行点击事件
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {//点击事件runable放到消息队列里执行
                                    performClickInternal();//放入消息队列失败，手动执行 
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }
                        //如果是Tap操作
                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }
                        //移除tap回调
                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;
                case MotionEvent.ACTION_CANCEL:
                //非探讨重点，暂时忽略
                case MotionEvent.ACTION_MOVE:
                 //非探讨重点，暂时忽略
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
                 case MotionEvent.ACTION_DOWN:
                //忽略    
            }

            return true;
        }

        return false;
    }
```

##### PerformClick
执行click的用于消息队列的runnalbe
```java
private final class PerformClick implements Runnable {
        @Override
        public void run() {
            performClickInternal();
        }
    }
```
##### performClickInternal

```java
  private boolean performClickInternal() {
        //点击事件通知自动填充manager
        notifyAutofillManagerOnClick();
        //执行真实点击事件
        return performClick();
    }
```
#####  performClick
```java
public boolean performClick() {
        //当点击通知自动填充manager
        notifyAutofillManagerOnClick();
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);//响应点击事件
            result = true;
        } else {
            result = false;
        }
        //通知辅助功能
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

        notifyEnterOrExitForAutoFillIfNeeded(true);

        return result;
}    
```

### viewGroup的分发方法讲解

#### viewGroup dispatchTouchEvent简析
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    
        //部分忽略
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {//过滤非正常情况，上文有讲解
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
            //部分源代码忽略
            final boolean intercepted;//用于viewGroup的拦截 分发的请求直接可以截断不向下下发，而是直接在本层进行处理
            if (actionMasked == MotionEvent.ACTION_DOWN //2.一般情况下，为初始 down和down后面mFirstTouchTarget不为空都会触发的
                    || mFirstTouchTarget != null) {//b.action为 move or up且有子view时会进入，因为 mFirstTouchTarget不为空
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);//执行拦截回调
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {//什么情况下 no touch targets 且不是初始第一次按下？
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            //部分源代码忽略

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            /*
				intercepted = true时，
				if (mFirstTouchTarget == null) {//此对象为空
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);//调用本viewGroup的super的view的方法，即将结果handled进行了赋值，
                        在后面的 move和up事件过来时，直接当作普通view调用

                intercepted = false时，
                action down move up都会进入该判断
            }
            */
             //部分忽略
             // 1.d. 有child的情况，都不会进入
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {//当 action 为down dispatchTransformedTouchEvent分发的结果为 false ，开始当作普通view执行
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
               //当ACTION_UP或ACTION_MOVE,mFirstTouchTarget已经被赋值过了，即不为空
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    //e. 当 action move up  alreadyDispatchedToNewTouchTarget = false 不执行
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        //当action 为 down 事件分发给child，则 alreadyDispatchedToNewTouchTarget = true
                        //此时 target == newTouchTarget 是成立的 就在 down中调用 addTouchTarget（）使得成立的
                        handled = true;//有子view的已经处理完毕
                    } else {//f. 当 action move up
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        //通过循环，直接将 down找到的touchTarget对象中保存的view,一一分发对应的move和up事件
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {//为true时就直接将链表向前移动，进行回收掉
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState(); //将FristTouchTarget置空
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        //部分忽略
        return handled;//返回结果
    }
```

### 简单概括

在我们很多针对的事件分发的讨论中，其实主要是针对ACTION_DOWN的一个流向的一个分析，但是根据实际分析可以发现，在ACTION_UP和ACTION_MOVE的情况下，代码执行的流程不同的：
不同主要体现在两个点：

a. View部分:

1. view的onTouEvent()部分,针对不同ACTION做的不同处理

```java
  if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                  case MotionEvent.ACTION_UP:
                  break;
            }
```

b. ViewGroup部分：

1. 在dispatchTouchEvent 当中对ACTION_UP做的处理主要就是，排除被拦截的情况外；主要就是直接将 **mFirstTouchTarget** 中的对应接收事件的child去直接取出来，将事件分发给对应的view 

## 总结
本文目前主要针对的Action_UP的情况的一个重点分析，接下来会补充已下内容：
1. 多指触控的事件分发指派
2. 对事件感兴趣和不感兴趣的流程图展示
