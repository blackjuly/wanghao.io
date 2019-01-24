---
layout:     post
title:      "Android touch Event Dispatch"
subtitle:   "Android 事件分发（理论下篇—— ACTION_MOVE ACTION_UP 发生了什么）"
date:       2018-11-20 21:36:00
author:     "wanghao"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - 源码
    
---

# Android 事件分发—— ACTION_UP,ACTION_MOVE 发生了什么


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

      //部分源代码忽略
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {//a.可以点击的情况情况；b.有悬停和长按显示工具提示框
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

##### mTouchDelegate.onTouchEvent
此为一个触控的代理类，处理一些特殊场景;最常见的应用场景，就是处理 **拓展view的触控区域**[**Extend a child view's touchable area官方文档链接**](https://developer.android.com/training/gestures/viewgroup#java) ，某一些情况下，我们的view展示必须要比较小，比如一个返回按钮 imageViewButton，但是会造成用户点击很难点击到，所以此时可以继承 TouchDelegate类，处理接收更大区域的点击事件
```java
if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
```
##### performButtonActionOnTouchDown

```java
protected boolean performButtonActionOnTouchDown(MotionEvent event) {
    if (event.isFromSource(InputDevice.SOURCE_MOUSE) &&
        (event.getButtonState() & MotionEvent.BUTTON_SECONDARY) != 0) {
            // 如果是鼠标右键,手写笔第一个按钮（详见BUTTON_SECONDARY常量注释）
        showContextMenu(event.getX(), event.getY());
        mPrivateFlags |= PFLAG_CANCEL_NEXT_UP_EVENT;
        return true;
    }
    return false;
}
```

### viewGroup的分发方法讲解

>dispatchTouchEvent方法分析
查看源码前，我们需要先了解一下 TouchTarget

#### TouchTarget简析
TouchTarget 是一个viewGroup中内部类，主要描述了触摸对应的child以及所捕获的数目不定的手指的ids
```java
private static final class TouchTarget {
        // 链表最大可回收的长度
        private static final int MAX_RECYCLED = 32;
         // 用于控制同步的锁
        private static final Object sRecycleLock = new Object[0];
        //  ViewGroup 中维护着一个 mFirstTouchEvent, 它是外部记录正在响应事件 View 的链表, 响应完成之后会调用 recycler 方法, 加入 sRecycleBin 这个可复用的链表中
        private static TouchTarget sRecycleBin;
        //内部可复用的链表长度
        private static int sRecycledCount;

        public static final int ALL_POINTER_IDS = -1; // all ones
        public View child;//被触控view
        // 手指id组合位掩码
        public int pointerIdBits;
        // 使用链表的数据结构，指向下一个触控对象
        public TouchTarget next;

        private TouchTarget() {
        }

        public static TouchTarget obtain(@NonNull View child, int pointerIdBits) {
            if (child == null) {
                throw new IllegalArgumentException("child must be non-null");
            }

            final TouchTarget target;
            synchronized (sRecycleLock) {
                if (sRecycleBin == null) {
                    target = new TouchTarget();
                } e// 从链表复用池中取出一个对象, 并重至属性值
                    target = sRecycleBin; // 将当前的表头赋给这个变量
                    sRecycleBin = target.next; // 表头移动到下个位置
                    sRecycledCount--; // 当前复用池的数量 -1 
                    target.next = null; // 将它的 next 置空
                }
            }
            target.child = child;//新的数据进行重新指定
            target.pointerIdBits = pointerIdBits;
            return target;
        }

        public void recycle() {
            if (child == null) {
                throw new IllegalStateException("already recycled once");
            }

            synchronized (sRecycleLock) {
                if (sRecycledCount < MAX_RECYCLED) {//小于回收长度
                    next = sRecycleBin;//将当前对象的回收对象赋值下一个
                    sRecycleBin = this;//自己设置为回收对象
                    sRecycledCount += 1;//回收池数字加1
                } else {
                    next = null;//超过回收长度，就释放掉下一个
                }
                child = null;//view置空
            }
        }
    }
```
#### viewGroup dispatchTouchEvent简析
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    
        //部分忽略
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {//过滤非正常情况，上文有讲解
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                //1.当MotionEvent.ACTION_DOWN进入事件，算是一个事件分发的开端
                //a.当MotionEvent.ACTION_UP 或move不进入事件

                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);//清楚所有 TouchTarget
                resetTouchState();//释放 mFirstTouchTarget的链表之前的所有引用，回收置空，方便不影响下一次
            }//目前，对TouchTarget的理解 数据结构为链表，由用于关联view和pointerId的对像组成的一组链表，同时方便down后面的事件，向下分发的时候
            //可以快速找到分发对象

            // Check for interception.
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

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {//忽略
                ev.setTargetAccessibilityFocus(false);
            }

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
            if (!canceled && !intercepted) {//canceled暂时不考虑; intercepted = false时，进入判断
                //部分忽略
                //1.action = down 进入
                //c. action = up 或 move 不进入
                if (actionMasked == MotionEvent.ACTION_DOWN//当动作为down
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)//暂时不清楚，推测是多指情况下，用到的
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);//移除指定的 手指id从targets的链表中

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        //目前猜测 MotionEvent.ACTION_DOWN
                        // ，才会进入该判断
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();//获取由z轴排序的子view的顺序列表
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();//此种情况为，自定义顺序的情况
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(//一般情况就是默认的index
                                    childrenCount, i, customOrder);
                            //只考虑正常情况下，preorderedList不为空,且取到的view不为空，验证过后直接返回对应
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                           //部分忽略

                            if (!canViewReceivePointerEvents(child)//忽略
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            // 当只考虑MotionEvent.ACTION_DOWN时，mFirstTouchTarget == null ,即 newTouchTarget会为null
                            //当不为MotionEvent.ACTION_DOWN时，需要从链表获取到对应 target（MotionEvent.ACTION_POINTER_DOWN，MotionEvent.ACTION_HOVER_MOVE）
                            if (newTouchTarget != null) {// MotionEvent.ACTION_DOWN 不进入，其他情况
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;//指定给新指针
                                break;
                            }

                            resetCancelNextUpFlag(child);//忽略
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {//分发结果 handler
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();//记录时间 x,y，index
                                //添加创建TouchTarget作为当前 fristTouchTarget的头节点
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                            //部分忽略
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }
                    //执行down时 newTouchTarget 肯定不为空，执行该段
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }
             // 1.d. 有child的情况，都不会进入
            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {//当 action 为down dispatchTransformedTouchEvent分发的结果为 false ，开始当作普通view执行
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    //e. 当 action move up  alreadyDispatchedToNewTouchTarget = false 不执行
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        //当action 为 down 事件分发给child，则 alreadyDispatchedToNewTouchTarget = true
                        //此时 target == newTouchTarget 是成立的 就在 down中调用 addTouchTarget（）使得成立的
                        handled = true;//有子view的 action为down的情况浏览完毕
                    } else {//f. 当 action move up
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        //通过循环，直接将 down找到的touchTarget对象中保存的view,一一分发 up和down事件对象
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
处理分发事件的重头，主要在 dispatchTransformedTouchEvent
##### dispatchTransformedTouchEvent
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
        //开始具体针对一个view子孩子的，事件分发 ，只有在down pointerDown hoverDown 进入该方法
        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {//cancel== true 需要读ACTION_CANCEL忽略阅读
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits(); //计算发现的指头数
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {//可能产生多余的事件却没有指头，要下掉该事件
            return false;
        }

        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {//对于多指情况下，暂时忽略
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {//没有孩子的情况，把当前viewGroup当做一个view调取本身的时间分发
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }
            //不为空，分发给子孩子
            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;//处理过
    }
```

## 总结
本文目前主要针对的Action_Down的情况的一个重点分析，接下来会补充已下内容：
1. viewGourp更多细节描述
2. 对事件感兴趣和不感兴趣的流程图展示
3. 对于Action_Up,Action_cancel等在下一篇博文，进行一个补充性详细说明