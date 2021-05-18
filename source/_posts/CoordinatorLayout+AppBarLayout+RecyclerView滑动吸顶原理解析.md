---
title: CoordinatorLayout+AppBarLayout+RecyclerView滑动吸顶原理解析
copyright: true
date: 2021-05-13 23:25:15
tags:
categories: Android UI
---



​	本文主要介绍使用`AppBarLayout+RecyclerView`滑动吸顶布局，其滑动原理，以及如何禁止滑动，包括禁止滑动`AppBarLayout`本身滑动，以及禁止滑动`RecyclerView`而让`AppBarLayout`滑动的操作。

<!-- more --> 

​	先上图

![图片](https://raw.githubusercontent.com/sunytan/sunytan.github.io.source/main/source/_posts/CoordinatorLayout%2BAppBarLayout%2BRecyclerView%E6%BB%91%E5%8A%A8%E5%90%B8%E9%A1%B6%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/stickerbar_demo.gif)

## 嵌套滑动原理

​	要达到如上效果，我们首先要知道其滑动原理，其主要还是使用了`NestedScrolling`的原理，`CoordinatorLayout`和`AppBarLayout`以及`RecyclerView`都实现了`NestedScrolling`的接口。主要是`NestedScrollingChildHelper`类和`NestedScrollingParent2`接口配合实现。还有`NestedScrollingParent3`只是在更高版本上的一些优化，此次不展开，有兴趣可以自行了解。

`NestedScrollingChildHelper`：子view的嵌套滑动代理类，实现对其父View的嵌套滑动操作。

`NestedScrollingParent2`：父view实现此接口，响应子view的嵌套滑动操作。

所以如果把`RecyclerView`换成`ListView`就无法达到滑动滑动吸顶的效果。因为`ListView`并没有实现此接口

### NestedScrolling接口

​	我们可以看下`NestedScrollingParent2`接口：

```java
/**
 * 由ViewGroup的子类实现，支持滑动操作能被其嵌套的子view代理
 * 实现这个接口的类，需要创建一个final的NestedScrollingParentHelper实例
 */
public interface NestedScrollingParent2 extends NestedScrollingParent {
  
  /**
   * 
   * 当前嵌套滑动的开始，如果父View接受嵌套滑动，就需要返回true，否者此轮滑动仅仅子View自己处理
   * 开始滑动，返回true，才能滑动，才会有后续的回调
   */
  boolean onStartNestedScroll(@NonNull View child, @NonNull View target, @ScrollAxis int axes,
            @NestedScrollType int type);
  
  /**
   * 接受了本轮嵌套滑动的回调通知，父View可以在接受后做自己的一些事情。
   */
  void onNestedScrollAccepted(@NonNull View child, @NonNull View target, @ScrollAxis int axes,
            @NestedScrollType int type);
  
  /**
   * 停止滑动的回调，停止后调用
   */
  void onStopNestedScroll(@NonNull View target, @NestedScrollType int type);
  
  /**
   * 子View本身滑动之后，如果还有剩余的未消费的距离，交给父View进行嵌套滑动处理。
   */
  void onNestedScroll(@NonNull View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, @NestedScrollType int type);
  
  /**
   * 子View本身滑动之前，预先交给父View处理嵌套滑动，父View把自身滑动消费的距离传回给子View
   * 子View再计算剩余还未消费的距离，然后子View再自己滑动剩下的距离
   */
  void onNestedPreScroll(@NonNull View target, int dx, int dy, @NonNull int[] consumed,
            @NestedScrollType int type);
}
```

### RecyclerView嵌套滑动实现

​	`RecyclerView`的滑动实现主要在于其`onTouchEvent`方法：

```java
@Override
public boolean onTouchEvent(MotionEvent e) {
  
  // ... 此处省略N行代码
  switch (action) {
            case MotionEvent.ACTION_DOWN: {
                mScrollPointerId = e.getPointerId(0);
                mInitialTouchX = mLastTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY() + 0.5f);

                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
              // 1.第一步
              // 开始嵌套滑动，初始化本次嵌套滑动的父View嵌套滑动状态。
              // 最终由NestedScrollingChildHelper.startNestedScroll代理方法实现其逻辑
                startNestedScroll(nestedScrollAxis, TYPE_TOUCH);
            } break;

            case MotionEvent.ACTION_POINTER_DOWN: {
                mScrollPointerId = e.getPointerId(actionIndex);
                mInitialTouchX = mLastTouchX = (int) (e.getX(actionIndex) + 0.5f);
                mInitialTouchY = mLastTouchY = (int) (e.getY(actionIndex) + 0.5f);
            } break;

            case MotionEvent.ACTION_MOVE: {
              
                // ... 此处省略N行代码
                if (mScrollState == SCROLL_STATE_DRAGGING) {
                    mReusableIntPair[0] = 0;
                    mReusableIntPair[1] = 0;
                  // 2.第二步
                  // 分发嵌套预滑动，传入mReusableIntPair数组。
                  // 数组记录被嵌套父view消费的滑动距离，最后返回出来。以此来计算自己需要滑动的距离
                  // 此处的父View其实就是CoordinatorLayout
                    if (dispatchNestedPreScroll(
                            canScrollHorizontally ? dx : 0,
                            canScrollVertically ? dy : 0,
                            mReusableIntPair, mScrollOffset, TYPE_TOUCH
                    )) {
                        dx -= mReusableIntPair[0];
                        dy -= mReusableIntPair[1];
                        // Updated the nested offsets
                        mNestedOffsets[0] += mScrollOffset[0];
                        mNestedOffsets[1] += mScrollOffset[1];
                        // Scroll has initiated, prevent parents from intercepting
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }

                    mLastTouchX = x - mScrollOffset[0];
                    mLastTouchY = y - mScrollOffset[1];

                  // 3.第三步
                  // RecyclerView内部自己滑动，自己滑动上一步的嵌套预滑动之后剩下的滑动距离。
                  // 如果全被外部父view消费掉，则自己无法滑动。
                  // 同样，其实里面还有判断如果RecyclerView以及滑动到边界了，同样还要分发给父view处理。
                    if (scrollByInternal(
                            canScrollHorizontally ? dx : 0,
                            canScrollVertically ? dy : 0,
                            e)) {
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }
                    if (mGapWorker != null && (dx != 0 || dy != 0)) {
                        mGapWorker.postFromTraversal(this, dx, dy);
                    }
                }
            } break;
				// ... 此处省略N行代码
        }
  
}
```

* 首先看第一步`startNestedScroll`的实现：

```java
    @Override
    public boolean startNestedScroll(int axes) {
      	// 就如前面所说就是用NestedScrollingChildHelper这个代理真正实现的。
        return getScrollingChildHelper().startNestedScroll(axes);
    }
```

​	其就是`NestedScrollingChildHelper`这个代理真正来实现的：

```java
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
        if (hasNestedScrollingParent(type)) {
            // Already in progress
            return true;
        }
  	// 1.此处由RecyclerView自己设置的是否允许嵌套滑动。
  	// 可以通过此处设置禁止嵌套滑动，从而实现只滑动RecyclerView本身
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent(); // 此处ViewParent就是CoordinatorLayout
            View child = mView; // RecyclerView本身
            while (p != null) {
              	// 2.此处其实就是交由父View来处理，返回值代表是否开始此轮的嵌套滑动
              	// 其实就是父View对NestedScrollingParent2接口的实现
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                  // 如果父View接受此次嵌套滑动，就设置此轮的嵌套滑动父View实例
                    setNestedScrollingParentForType(type, p);
                  // 回调父View接受了此次嵌套滑动
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
```

​	从上面的代码注释已经能看出了，两种禁止嵌套滑动的方式。

 	1. 通过RecyclerView.setNestedScrollingEnabled方法
 	2. 通过重写父View的onStartNestedScroll方法让其返回false

* 接下来看第二步`dispatchNestedPreScroll`方法：

  其实就是`NestedScrollingChildHelper`的`dispatchNestedPreScroll`来实现的

```java
    public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
            @Nullable int[] offsetInWindow, @NestedScrollType int type) {
      // 此和上面一样就是RecyclerView中设置是否允许嵌套滑动
        if (isNestedScrollingEnabled()) {
          	// 获取嵌套滑动的父View，在startNestedScroll中已经赋值过，此处就是CoordinatorLayout
            final ViewParent parent = getNestedScrollingParentForType(type);
            if (parent == null) {
                return false;
            }

            if (dx != 0 || dy != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                if (consumed == null) {
                    consumed = getTempNestedScrollConsumed();
                }
                consumed[0] = 0;
                consumed[1] = 0;
              // 此处就是真正的父view处理预滑动，通过consumed这个数组传递，来得到父view消费掉的距离
              // 使用数组对象传递而不是用的所谓返回值。也一样带出结果
                ViewParentCompat.onNestedPreScroll(parent, mView, dx, dy, consumed, type);

                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return consumed[0] != 0 || consumed[1] != 0;
            } else if (offsetInWindow != null) {
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
```

代码中关键步骤注释都写的很清楚了。

**这个方法其实就一个用处：** 调用父view的`onNestedPreScroll`接口通过传入的数组来得到父view消费了多少滑动距离。

* 再接下来看下第三步其内部滑动`scrollByInternal` ：

```java
    boolean scrollByInternal(int x, int y, MotionEvent ev) {
        int unconsumedX = 0;
        int unconsumedY = 0;
        int consumedX = 0;
        int consumedY = 0;

        consumePendingUpdateOperations();
        if (mAdapter != null) {
            mReusableIntPair[0] = 0;
            mReusableIntPair[1] = 0;
          // RecyclerView自己的滑动步骤，使用mReusableIntPair数组来记录滑动消费的距离
            scrollStep(x, y, mReusableIntPair);
          // 得到x,y两个方向的消费距离
            consumedX = mReusableIntPair[0];
            consumedY = mReusableIntPair[1];
          // 剩下的未消费的距离
            unconsumedX = x - consumedX;
            unconsumedY = y - consumedY;
        }
        if (!mItemDecorations.isEmpty()) {
            invalidate();
        }

        mReusableIntPair[0] = 0;
        mReusableIntPair[1] = 0;
      // 分发嵌套滑动，再RecyclerView自己滑动之后，剩下的滑动距离再次分发给父view
        dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset,
                TYPE_TOUCH, mReusableIntPair);
      // 同样通过mReusableIntPair数组得到父view滑动消费的距离
        unconsumedX -= mReusableIntPair[0];
        unconsumedY -= mReusableIntPair[1];
        boolean consumedNestedScroll = mReusableIntPair[0] != 0 || mReusableIntPair[1] != 0;

        // Update the last touch co-ords, taking any scroll offset into account
        mLastTouchX -= mScrollOffset[0];
        mLastTouchY -= mScrollOffset[1];
        mNestedOffsets[0] += mScrollOffset[0];
        mNestedOffsets[1] += mScrollOffset[1];

        if (getOverScrollMode() != View.OVER_SCROLL_NEVER) {
            if (ev != null && !MotionEventCompat.isFromSource(ev, InputDevice.SOURCE_MOUSE)) {
                pullGlows(ev.getX(), unconsumedX, ev.getY(), unconsumedY);
            }
            considerReleasingGlowsOnScroll(x, y);
        }
        if (consumedX != 0 || consumedY != 0) {
          // RecyclerView的自己滑动，分发
            dispatchOnScrolled(consumedX, consumedY);
        }
        if (!awakenScrollBars()) {
            invalidate();
        }
        return consumedNestedScroll || consumedX != 0 || consumedY != 0;
    }
```

​	可以看上面代码注释，此方法主要作用作用：

	1. 先滑动`RecyclerView`本身。
 	2. 然后把剩余的未消费滑动距离再分发给到父view进行滑动。

### 子`RecyclerView`嵌套滑动总结

​	从上面可以看到嵌套滑动的子View的整体嵌套滑动步骤分为三大步：

**第一步:** `startNestedScroll`

​	开启本轮嵌套滑动，调用父view实现的`NestedScrollingParent2.onStartNestedScroll`接口

**第二步:**  `dispatchNestedPreScroll` 

​		分发嵌套预滑动，把滑动距离预先分发给父view，父view自己处理决定是否消费。

​		这里主要是调用了父类实现的``NestedScrollingParent2.onNestedPreScroll` 接口

​		所以滑动`RecyclerView`的时候，先响应头还是先响应本身正是基于此方法实现的。

**第三步:** `scrollByInternal`

​		`RecyclerView`自己内部滑动未被父view消费的距离。

​		自身滑动后，再把还剩下的未消费的距离再次分发给到父布局。父布局可以再次进行消费。

​		这里主要调用父类实现的 `NestedScrollingParent2.onNestedScroll`

到此嵌套滑动`NestedScrollingParent2`的滑动接口以及全部用到了。

### 父`CoordinatorLayout`嵌套滑动实现

​	上面以及完整讲过子`RecyclerView`嵌套滑动的整个步骤，其最终都会调用到父view的嵌套滑动接口。接下来在按照其先后顺序看下父view的嵌套滑动的接口实现。

​	首先来看下`onStartNestedScroll`的实现

```java
@Override
@SuppressWarnings("unchecked")
public boolean onStartNestedScroll(View child, View target, int axes, int type) {
    boolean handled = false;
  
  // 由此其实父布局CoordinatorLayout并不是滑动本身，而是继续滑动其所有子布局
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
      // 不可见的子布局不参与嵌套滑动
        if (view.getVisibility() == View.GONE) {
            // If it's GONE, don't dispatch
            continue;
        }
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
      // 其子view布局中必须实现layout_behavior，才能实现嵌套滑动
      // 所以本例中的悬浮按钮因为没有实现behavior，不会跟随嵌套滑动。
        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
          // 调用其behavior中的onStartNestedScroll，根据其不同的行为决定当前子view是否接受嵌套滑动
          // 所以AppBarLayout的嵌套滑动完全交由AppBarLayout.Behavior处理，本例中使用了自定义的MyBehavior
            final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child,
                    target, axes, type);
            handled |= accepted;
            lp.setNestedScrollAccepted(type, accepted);
        } else {
            lp.setNestedScrollAccepted(type, false);
        }
    }
    return handled;
}
```

从上又有一种禁止嵌套滑动的的方法

1 .  自定义`AppBarLayout`的Behavior，使`onStartNestedScroll` 返回false，则此子View不会进行嵌套滑动。

接下来再来看`onNestedPreScroll`：

```java
@Override
@SuppressWarnings("unchecked")
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed, int  type) {
    int xConsumed = 0;
    int yConsumed = 0;
    boolean accepted = false;

  // 由此其实父布局CoordinatorLayout并不是滑动本身，而是继续滑动其所有子View
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        if (view.getVisibility() == GONE) {
            // If the child is GONE, skip...
            continue;
        }

        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
      // 此处onStartNestedScroll中进行了赋值，判断是否接受嵌套滑动
        if (!lp.isNestedScrollAccepted(type)) {
            continue;
        }

        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            mBehaviorConsumed[0] = 0;
            mBehaviorConsumed[1] = 0;
          // 子view的嵌套滑动完全交由其Behavior处理，本例也就是AppBarLayout.Behavior
            viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mBehaviorConsumed, type);

            xConsumed = dx > 0 ? Math.max(xConsumed, mBehaviorConsumed[0])
                    : Math.min(xConsumed, mBehaviorConsumed[0]);
            yConsumed = dy > 0 ? Math.max(yConsumed, mBehaviorConsumed[1])
                    : Math.min(yConsumed, mBehaviorConsumed[1]);

            accepted = true;
        }
    }

  // 此处就是数组存储已经嵌套滑动预先消费掉的距离，通过此数组对象带回去给了RecyclerView的mReusableIntPair
    consumed[0] = xConsumed;
    consumed[1] = yConsumed;

    if (accepted) {
        onChildViewsChanged(EVENT_NESTED_SCROLL);
    }
}
```

​	从上述代码可以看出，在父View的`onNestedPreScroll`中，主要是调用子类的Behavior的`onNestedPreScroll`方法，由子类的Behavior真正实现如何响应滑动，子View进行滑动后会把自己消费了多少距离，通过数组传回给到了最初的子View也就是`RecyclerView`

​	其实对于AppBarLayout如何嵌套滑动完全都可以由我们自己自定义。

再接下来我们看下`onNestedScroll`

```java
    @Override
    @SuppressWarnings("unchecked")
    public void onNestedScroll(@NonNull View target, int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, @ViewCompat.NestedScrollType int type,
            @NonNull int[] consumed) {
        final int childCount = getChildCount();
        boolean accepted = false;
        int xConsumed = 0;
        int yConsumed = 0;

      // 整个处理过程其实跟onNestedPreScroll一样。
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted(type)) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {

                mBehaviorConsumed[0] = 0;
                mBehaviorConsumed[1] = 0;

                viewBehavior.onNestedScroll(this, view, target, dxConsumed, dyConsumed,
                        dxUnconsumed, dyUnconsumed, type, mBehaviorConsumed);

                xConsumed = dxUnconsumed > 0 ? Math.max(xConsumed, mBehaviorConsumed[0])
                        : Math.min(xConsumed, mBehaviorConsumed[0]);
                yConsumed = dyUnconsumed > 0 ? Math.max(yConsumed, mBehaviorConsumed[1])
                        : Math.min(yConsumed, mBehaviorConsumed[1]);

                accepted = true;
            }
        }

        consumed[0] += xConsumed;
        consumed[1] += yConsumed;

        if (accepted) {
            onChildViewsChanged(EVENT_NESTED_SCROLL);
        }
    }
```

​	从上述代码可以看到`onNestedScroll`和`onNestedPreScroll`的处理过程基本一样，本来也都是处理`AppBarLayout`的滑动，只是处理的时机不一样罢了。

到此整个嵌套滑动的过程以及很清晰了。

下面附上时序图：

![时序图](https://raw.githubusercontent.com/sunytan/sunytan.github.io.source/main/source/_posts/CoordinatorLayout%2BAppBarLayout%2BRecyclerView%E6%BB%91%E5%8A%A8%E5%90%B8%E9%A1%B6%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

### 总结

​	以上是以`RecyclerView`为例，讲述的原理流程，其实也可以替换成任何实现了嵌套滑动接口的View。关于如何禁止滑动其实在过程中已经讲的很清楚了。没有再细讲如何实现吸顶的，其实想想也能就能明白，吸顶其实就是上滑的时候，滑到一定距离就不让`AppBarLayout`上移了。那其实就是可以通过其本身滑动的Behavior控制其可滑动距离罢了。当然其实`AppBarLayout`本身已经实现好了，其通过给其子View设置`layout_scrollFlags`来控制其滑动行为。这个在此不展开了。



本地使用示例代码地址：[https://github.com/sunytan/StickerBarDemo/](https://github.com/sunytan/StickerBarDemo/)

