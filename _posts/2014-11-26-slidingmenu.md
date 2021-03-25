---
layout:     post
title:      "SlidingMenu源码分析"
subtitle:   ""
date:       2014-11-26 12:00:00
author:     "boredream"
catalog: false
header-style: text
tags:
  - 源码

---


研究SlidingMenu之前最好先了解自定义组件相关的知识，以及简单看下我之前文章里的ScrollView原理分析~


SlidingMenu设计思路
三个主要的ViewGroup, 主ViewGroup里面包含两个重叠的ViewGroup,盖在上面的就是显示内容,而下面的就是菜单
上面的侧滑以后露出后面的View~
主ViewGroup作为一个自定义控件,里面的内容和菜单利用自定义属性设置


SlidingMenu(继承RelativeLayout),就是这个主ViewGroup,
其中又包含两个,内容CustomViewAbove和菜单CustomViewBehind(都继承ViewGroup),
SlidingMenu提供两个自定义属性,供使用者传入两个布局id,对应主体内容和菜单布局,
此外还有其他一些属性供开发者设置菜单效果

---

代码分析
构造函数中代码如下:

```java
LayoutParams behindParams = new LayoutParams(LayoutParams. MATCH_PARENT, LayoutParams.MATCH_PARENT);
mViewBehind = new CustomViewBehind(context);
addView(mViewBehind , behindParams);
LayoutParams aboveParams = new LayoutParams(LayoutParams. MATCH_PARENT, LayoutParams.MATCH_PARENT);
mViewAbove = new CustomViewAbove(context);
addView(mViewAbove , aboveParams);
// register the CustomViewBehind with the CustomViewAbove
mViewAbove.setCustomViewBehind(mViewBehind );
mViewBehind.setCustomViewAbove(mViewAbove );
mViewAbove.setOnPageChangeListener(new OnPageChangeListener() {
      public static final int POSITION_OPEN = 0;
      public static final int POSITION_CLOSE = 1;
      public static final int POSITION_SECONDARY_OPEN = 2;

      public void onPageScrolled( int position, float positionOffset,
                   int positionOffsetPixels) { }

      public void onPageSelected( int position) {
             if (position == POSITION_OPEN && mOpenListener != null) {
                   mOpenListener.onOpen();
            } else if (position == POSITION_CLOSE && mCloseListener != null) {
                   mCloseListener.onClose();
            } else if (position == POSITION_SECONDARY_OPEN && mSecondaryOpenListner != null ) {
                   mSecondaryOpenListner.onOpen();
            }
      }
});

// now style everything!
TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.SlidingMenu);
// set the above and behind views if defined in xml
int mode = ta.getInt(R.styleable.SlidingMenu_mode, LEFT);
setMode(mode);
int viewAbove = ta.getResourceId(R.styleable. SlidingMenu_viewAbove, -1);
if (viewAbove != -1) {
      setContent(viewAbove );
} else {
      setContent( new FrameLayout(context));
}
int viewBehind = ta.getResourceId(R.styleable.SlidingMenu_viewBehind, -1);
if (viewBehind != -1) {
      setMenu(viewBehind);
} else {
      setMenu(new FrameLayout(context));
}
```

新建两个自定义控件CustomViewAbove/Behind将其addView到SlidingMenu中,
然后在通过属性获取到view的id后会在setContent/Menu方法中将view和layout绑定

下面是content,menu同理

```java
/**
 * Set the above view content to the given View.
 *
 * @param view The desired content to display.
 */
public void setContent(View view) {
      mViewAbove.setContent(view);
      showContent();
}

public void setContent(View v) {
      if (mContent != null)
             this.removeView( mContent);
      mContent = v;
      addView(mContent);
}
```

同时,让当前显示content部分showContent(), 内部是调用了setCurrentItem方法如下,会在后面介绍

```java
/**
 * Closes the menu and shows the above view.
 *
 * @param animate true to animate the transition, false to ignore animation
 */
public void showContent(boolean animate) {
      mViewAbove.setCurrentItem(1, animate);
}
```

下面分别介绍Above和Behind两部分

---

Above
CustViewAbove类,包含主要内容的类,即显示在SlidingMenu上面/前端的部分
由于SlidingMenu实现效果重点在于滑动,滑动前端显示的内容然后展现出下面的菜单部分,
所以重中之重自然就是这个CustViewAbove类里的onTouchEvent里的处理了

以下是核心方法ouTouchEvent中的分析,请先看完教程 Scroll效果研究-系统ScrollView源码分析
SlidingMenu的CustomViewAbove和ScrollView处理没什么区别,看注释就知道很多都是直接拷贝过去的


同理分按下,滑动,抬起几部分看
1. 按下
```java
case MotionEvent.ACTION_DOWN :
      /*
       * If being flinged and user touches, stop the fling. isFinished
       * will be false if being flinged.
       */
      completeScroll();

      // Remember where the motion event started
      int index = MotionEventCompat. getActionIndex(ev);
      mActivePointerId = MotionEventCompat. getPointerId(ev, index);
      mLastMotionX = mInitialMotionX = ev.getX();
```

如果还在滚动则停止~
记住开始在哪里点下的,这里只记录了x轴坐标,因为侧滑只关注横线移动~
mActivityPointerId是记录多点触碰的activity第一个点的id,总之是用来处理多点除控时拖动的稳定性


2. 滑动
```java
case MotionEvent.ACTION_MOVE :
      if (!mIsBeingDragged) { 
            determineDrag(ev);
             if ( mIsUnableToDrag)
                   return false;
      }
      if (mIsBeingDragged) {
             // Scroll to follow the motion event
             final int activePointerIndex = getPointerIndex(ev, mActivePointerId);
             if ( mActivePointerId == INVALID_POINTER)
                   break;
             final float x = MotionEventCompat. getX(ev, activePointerIndex);
             final float deltaX = mLastMotionX - x;
             mLastMotionX = x;
             float oldScrollX = getScrollX();
             float scrollX = oldScrollX + deltaX;
             final float leftBound = getLeftBound();
             final float rightBound = getRightBound();
             if (scrollX < leftBound) {
                  scrollX = leftBound;
            } else if (scrollX > rightBound) {
                  scrollX = rightBound;
            }
             // Don't lose the rounded component
             mLastMotionX += scrollX - ( int) scrollX;
            scrollTo(( int) scrollX, getScrollY());
            pageScrolled(( int) scrollX);
      }
      break;
```
挑重点介绍了
首先是determineDrag方法,即确定当前动作是否算是一个我们需要的拖动事件,是否要消费处理之~
方法如下
```java
private void determineDrag(MotionEvent ev) {
      final int activePointerId = mActivePointerId;
      final int pointerIndex = getPointerIndex(ev, activePointerId);
      if (activePointerId == INVALID_POINTER || pointerIndex == INVALID_POINTER)
             return;
      final float x = MotionEventCompat. getX(ev, pointerIndex);
      final float dx = x - mLastMotionX;
      final float xDiff = Math.abs(dx);
      final float y = MotionEventCompat. getY(ev, pointerIndex);
      final float dy = y - mLastMotionY;
      final float yDiff = Math.abs(dy);
      if (xDiff > (isMenuOpen()? mTouchSlop/2: mTouchSlop) && xDiff > yDiff && thisSlideAllowed(dx)) {          
            startDrag();
             mLastMotionX = x;
             mLastMotionY = y;
            setScrollingCacheEnabled( true);
             // TODO add back in touch slop check
      } else if (xDiff > mTouchSlop) {
             mIsUnableToDrag = true;
      }
}
```
最主要是地方在于if 对diff的判断语句
其中x/yDiff是x和y轴的移动距离长度,
mTouchSlop是系统对于拖动的最低长度判断,即至少移动多少多少距离,才算是一个拖动的动作~

所以判断的条件就是:
当菜单没打开时,xDiff大于mTouchSlop时才视为一个我们所需要的拖动,打开时则只要一半即可,
并且要同时满足xDiff>yDiff,即拖动是一个横向大于45度的
thisSlideAllowed可以理解为是判断是否是在菜单打开时touch到了菜单部分的位置

总结起来就是
如果拖动距离达到最小阀值且大于横向45度角度,且是touch到主体部分,则为我们需要的触摸事件


然后记录x,y的赋值给mLastMotionX/Y

当determineDrag判断这是一个我们需要的拖动事件了以后,就开始让aboveView随着动作滚动了


首先是最后一行pageScrolled(( int) scrollX);设置监听数据的,先无视

最重要是scrollTo方法,在系统的scrollTo基础上又加了点其他处理,如下
```java
@Override
public void scrollTo(int x, int y) {
      super.scrollTo(x, y);
      mScrollX = x;
      mViewBehind.scrollBehindTo( mContent, x, y);     
      ((SlidingMenu)getParent()).manageLayers(getPercentOpen());
}
```
下面代码中manageLayers是提高性能用的,不深入研究了
super.scrollTo就是让aboveView随着手势拖动滚动,这个比较简单

此外,
使用SlidingMenu的时候可以注意到,滚动上面主体内容展示后面菜单时,菜单也有一个滚动展开效果
这个mViewBehind.scrollBehindTo( mContent, x, y);就是处理这个的
这里先知道作用,分析完above部分后再详细分析behind部分,会介绍这个作用



然后是抬起方法,对于SlidingMenu,如果使用过肯定会有一定了解,以下是一些效果需求
如果菜单打开到一定位置,则抬起时菜单会完全打开
如果菜单打开部分太小,则抬起时菜单会收回去
还要有甩的处理,随着手势甩开菜单,甩关闭菜单~


下面是代码部分
```java
case MotionEvent.ACTION_UP :
      if (mIsBeingDragged) {
             final VelocityTracker velocityTracker = mVelocityTracker;
            velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
             int initialVelocity = ( int) VelocityTrackerCompat.getXVelocity(
                        velocityTracker, mActivePointerId);
             final int scrollX = getScrollX();
             final float pageOffset = ( float) (scrollX - getDestScrollX(mCurItem)) / getBehindWidth();
             final int activePointerIndex = getPointerIndex(ev, mActivePointerId);
             if ( mActivePointerId != INVALID_POINTER) {
                   final float x = MotionEventCompat. getX(ev, activePointerIndex);
                   final int totalDelta = ( int) (x - mInitialMotionX);
                   int nextPage = determineTargetPage(pageOffset, initialVelocity, totalDelta);
                  setCurrentItemInternal(nextPage, true, true, initialVelocity);
            } else {    
                  setCurrentItemInternal( mCurItem, true, true, initialVelocity);
            }
             mActivePointerId = INVALID_POINTER;
            endDrag();
      } else if (mQuickReturn && mViewBehind.menuTouchInQuickReturn( mContent, mCurItem, ev.getX() + mScrollX)) {
             // close the menu
            setCurrentItem(1);
            endDrag();
      }
      break;
```
同样,简单的不介绍了,参考之前的ScrollView源码分析教程
直接定位到SlidingMenu中特殊处理的部分

pageOffset为菜单当前打开比例,是一个0~1的值,比如打开一半了比例就是0.5,
该值可以是正可以是负,用于表示方向,向右为正向左为负~
计算方法为
```java
final float pageOffset = ( float) (scrollX - getDestScrollX(mCurItem )) / getBehindWidth();
```
mCurItem为当前页索引,可能是0,1,2
0代表左边的菜单打开状态,1是没有菜单打开,2是右边菜单打开

计算出页pageOffset后再根据速度和移动距离最终算出目标页位置,方法如下
```java
private int determineTargetPage (float pageOffset, int velocity, int deltaX) {
      int targetPage = mCurItem;
      if (Math.abs(deltaX) > mFlingDistance && Math.abs(velocity) > mMinimumVelocity) {
             if (velocity > 0 && deltaX > 0) {
                  targetPage -= 1;
            } else if (velocity < 0 && deltaX < 0){
                  targetPage += 1;
            }
      } else {
            targetPage = ( int) Math. round(mCurItem + pageOffset);
      }
      return targetPage;
}
```
速度和距离达到最小值后,则根据速度和距离的正负控制当前页索引加或减1,
当达不到if条件时,则计算当前页索引mCurItem与菜单打开状态比例值pageOffset和的四舍五入值~

举个else处理的例子
比如当前页为1即未打开菜单状态,此时左移了菜单宽度十分之三的距离,即pageOffset=-0.3
那结果就是 1 - 0.3 = 0.7 取四舍五入就是1~ 当前页还是1,即速度不够时移动三分之一抬起菜单还会收回去
还是刚才的条件,左移换成了十分之八的距离,即pageOffset=-0.8,
结果就是1 - 0.8 = 0.2 四舍五入就是0~ 即当前页应该是0左菜单了

以上简单总结就是,速度距离足够后,根据速度控制左右滑动来显示菜单或者内容
如果速度不够,则移动超过菜单一半距离时就切换菜单打开关闭状态,不然就收回去


注意,determineTargetPage方法只是获取目标页位置,实际跳转工作是下面的方法
```java
void setCurrentItemInternal(int item, boolean smoothScroll, boolean always, int velocity) {
      if (!always && mCurItem == item) {
            setScrollingCacheEnabled( false);
             return;
      }

      item = mViewBehind.getMenuPage(item);

      final boolean dispatchSelected = mCurItem != item;
      mCurItem = item;
      final int destX = getDestScrollX(mCurItem );
      if (dispatchSelected && mOnPageChangeListener != null) {
             mOnPageChangeListener.onPageSelected(item);
      }
      if (dispatchSelected && mInternalPageChangeListener != null ) {
             mInternalPageChangeListener.onPageSelected(item);
      }
      if (smoothScroll) {
            smoothScrollTo(destX, 0, velocity);
      } else {
            completeScroll();
            scrollTo(destX, 0);
      }
}
```
监听设置无视,核心方法是smoothScrollTo方法,如下

```java
/**
 * Like {@link View#scrollBy}, but scroll smoothly instead of immediately.
 *
 * @param x the number of pixels to scroll by on the X axis
 * @param y the number of pixels to scroll by on the Y axis
 * @param velocity the velocity associated with a fling, if applicable. (0 otherwise)
 */
void smoothScrollTo(int x, int y, int velocity) {
      if (getChildCount() == 0) {
             // Nothing to do.
            setScrollingCacheEnabled( false);
             return;
      }
      int sx = getScrollX();
      int sy = getScrollY();
      int dx = x - sx;
      int dy = y - sy;
      if (dx == 0 && dy == 0) {
            completeScroll();
             if (isMenuOpen()) {
                   if ( mOpenedListener != null)
                         mOpenedListener.onOpened();
            } else {
                   if ( mClosedListener != null)
                         mClosedListener.onClosed();
            }
             return;
      }

      setScrollingCacheEnabled( true);
      mScrolling = true;

      final int width = getBehindWidth();
      final int halfWidth = width / 2;
      final float distanceRatio = Math. min(1f, 1.0f * Math.abs(dx) / width);
      final float distance = halfWidth + halfWidth *
                  distanceInfluenceForSnapDuration(distanceRatio);

      int duration = 0;
      velocity = Math.abs(velocity);
      if (velocity > 0) {
            duration = 4 * Math. round(1000 * Math.abs (distance / velocity));
      } else {
             final float pageDelta = ( float) Math. abs(dx) / width;
            duration = ( int) ((pageDelta + 1) * 100);
            duration = MAX_SETTLE_DURATION;
      }
      duration = Math.min(duration, MAX_SETTLE_DURATION);

      mScroller.startScroll(sx, sy, dx, dy, duration);
      invalidate();
}
```
最后发现内部实现最终还是Scroller类的startScroll方法~让它继续完成打开/关闭菜单的剩下动画
至于duration滑动持续时间的计算算法就不研究了~Scroller用法也不介绍了,之前ScrollView教程里有

---

以上部分其实看懂ScrollView原理以后看完全没压力理解,但还有一个没介绍的知识点,touch事件分发问题~

问题场景:
在菜单打开的时候要进行判断,
如果点击在主体部分,则自动收回菜单,即让主体部分消费这个touch事件,
如果此时点击在菜单部分,那above中应该不消费此事件,要将其传给behind部分,
然后让behind中的listview或者button等自定义设置的控件去获取处理touch事件~


此外还有菜单状态打开时点主体部分则直接关闭菜单,或者back键直接关闭菜单等,原理上面都介绍了,
这些逻辑方面的处理还有其他一些优化部分就不介绍了~

---

Behind
菜单部分CustomViewBehind中内容不多,主要有两大块内容

1. 滚动
研究主体内容部分的时候已经提到过,当前端内容部分拖动显示/关闭菜单时,菜单也有一个随之滚动的效果~

2. 效果绘制
包括阴影,淡入淡出效果等


绘制部分比较复杂需要以后有时间专门介绍,先介绍滚动相关的部分即1
相关方法如下,该方法在CustomViewAbove的scrollTo中调用
```java
public void scrollBehindTo(View content, int x, int y) {
      int vis = View. VISIBLE;
      if (mMode == SlidingMenu. LEFT) {
             if (x >= content.getLeft()) vis = View. INVISIBLE;
            scrollTo(( int)((x + getBehindWidth())* mScrollScale), y);
      } else if (mMode == SlidingMenu. RIGHT) {
             if (x <= content.getLeft()) vis = View. INVISIBLE;
            scrollTo(( int)(getBehindWidth() - getWidth() +
                        (x-getBehindWidth())* mScrollScale), y);
      } else if (mMode == SlidingMenu. LEFT_RIGHT) {
             mContent.setVisibility(x >= content.getLeft() ? View.INVISIBLE : View.VISIBLE );
             mSecondaryContent.setVisibility(x <= content.getLeft() ? View.INVISIBLE : View.VISIBLE );
            vis = x == 0 ? View. INVISIBLE : View. VISIBLE;
             if (x <= content.getLeft()) {
                  scrollTo(( int)((x + getBehindWidth())* mScrollScale), y);                     
            } else {
                  scrollTo(( int)(getBehindWidth() - getWidth() +
                              (x-getBehindWidth())* mScrollScale), y);                        
            }
      }
      if (vis == View. INVISIBLE)
            Log. v(TAG, "behind INVISIBLE" );
      setVisibility(vis);
}
```
只分析LEFT的情况(请他情况同理)
比如菜单的width即getBehindWidth的宽度为300
那从菜单关闭状态一直滚到菜单完全打开状态,above主体内容x轴上的scroll变化就是0 ~ -300
此时如果mScrollScale即滚动比例为0.3
那按照上面scrollBehindTo中的算法, behind菜单部分x轴上的scroll变化就是
(0+300)*0.3 ~ (-300+300)*0.3即100~0

如果比例换成0.6,那behind的x变化就是180~0

极端情况下
1. mScrollScale=0, behind的x变化为 0~0, 此时会发现behind菜单部分在打开关闭的过程中无任何滚动
2. mScrollScale=1,behind的换标为300~0, 此时的效果就是主体和菜单紧挨着以同一个速度滚动

注意,这里的x为滚动的偏移量,即scrollTo使用的参数,
比如scrollView中,scrollTo(0, 100)会往下滚动到y偏移量100的位置,
此时scrollView里面的内容视觉上看其实是向上移了~
这里同理
虽然朝右拖动的时候view看上去是朝右运动,但x轴上滚动偏移量是减少的



demo
BottomSlidingMenu 已备份至百度云盘
根据SlidingMenu建了一个底部划出的菜单效果,删除了SlidingMenu中大量代码,只保留了最核心的部分方便理解

缺陷
限定死了behind菜单部分的高度和底部运动的比例值等在SlidingMenu中是可以设定的信息

---

源码部分介绍结束
建议大家可以参考SlidingMenu简单的自己实现个底部划出的菜单练下手