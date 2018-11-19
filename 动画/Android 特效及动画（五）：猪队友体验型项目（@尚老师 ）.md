## 主要内容

* [项目介绍](#1)
* [涉及到的动画](#2)
    * [触摸反馈动画](#2.1) 
        * [定制触摸反馈动画](#2.1.1)
        * [波纹动画原理](#2.1.2)
    * [转场动画](#3)
        * [实现原理](#3.1) 
        * [注意点](#3.2) 

<h3 id="1">项目介绍</h3>

猪队友是一个游戏类的短视频应用，支持游戏中精彩片段的自动识别并保存、视频编辑上传。

本来想录制一个使用视频展示下的，但是现在域名好像已经废弃掉了。。。

所以只能讲一讲里面涉及和使用的动画。

<h3 id="2">涉及到的动画</h3>

由于屏幕录制截取是这个应用的基础功能，所以sdk mini version是5.0(21)，因此可以使用5.0以上的一些新的动画特性。

<h4 id="2.1">触摸反馈动画</h4>

触摸反馈动画其实在5.0之前就有的，只不过是一种静态的反馈，就是State list，给不同的状态定义不同的背景。

    <selector xmlns:android="http://schemas.android.com/apk/res/android">
        <item android:state_focused="true" android:drawable="@android:color/holo_green_dark"/>
        <item android:state_pressed="true" android:drawable="@android:color/holo_green_dark"/>
        <item android:drawable="@android:color/holo_blue_bright"/>
    </selector>
    
    
5.0之后是一种新的触摸反馈：波纹动画，较之前的State list，不仅有颜色的变化，还有一个持续扩散的波纹。

默认的波纹反馈：是以点击的地方为中心，有水波纹向四周扩散。


<h5 id="2.1.1">定制触摸反馈动画</h5>

大部分时候，默认的反馈动画是能满足我们的需求的，但是有时候需要一些特殊的定制满足产品的奇思妙想。

目前的定制只支持：动画颜色、波纹范围；不支持对波纹动画的改变

1. 定制波纹颜色

![image](http://pduvi7ojx.bkt.clouddn.com//18-8-30/56975289.jpg)

2. 定制波纹边界

    把控件的background属性的值设置成：
    * ?android:attr/selectableItemBackground 指定波纹有边界，即波纹只在控件的范围内，默认值
    * ?android:attr/selectableItemBackgroundBorderless 指定越过视图边界的波纹。 它将由一个非空背景的视图的最近父项所绘制和设定边界。API 级别 21 中推出的新属性
    
3. 使用xml实现

    使用修改主题的方式，会影响全局的样式，一般很少采用。
    
    通过在drawable目录下创建一个ripple文件：
    
        <?xml version="1.0" encoding="utf-8"?>
        <ripple xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:tools="http://schemas.android.com/tools"
            android:color="@color/colorPrimaryDark"
            android:radius="120dp"
            tools:ignore="NewApi">
            <!--没有item，表示波纹边界越过控件，类似： selectableItemBackgroundBorderless， 添加了item就会变成 selectableItemBackground -->
            <!--<item-->
                <!--android:id="@android:id/mask"-->
                <!--android:drawable="@color/colorPrimary"/>-->
        
        </ripple>

*注：一个可以设置波纹样式的库[https://github.com/traex/RippleEffect]*

<h5 id="2.1.2">波纹动画原理</h5>

**一眼看到这个动画，可以大概猜想它的实现应该是： 画圆 -> 同时添加透明度，半径变化的 属性动画。**

一个view有图像需要绘制时，会回调draw()方法，draw()方法再调用Drawable的draw()方法，把图像绘制出来。

波纹动画是由RippleDrawable负责绘制的，当点击一个控件时，调用RippleDrawable的draw()方法绘制波纹：

    @Override
    public void draw(@NonNull Canvas canvas) {
        pruneRipples();

        // Clip to the dirty bounds, which will be the drawable bounds if we
        // have a mask or content and the ripple bounds if we're projecting.
        final Rect bounds = getDirtyBounds();
        final int saveCount = canvas.save(Canvas.CLIP_SAVE_FLAG);
        if (isBounded()) {
            canvas.clipRect(bounds);
        }
        drawContent(canvas);
        drawBackgroundAndRipples(canvas);
        canvas.restoreToCount(saveCount);
    }

可以看到，如果有边界，会对canvas进行裁剪，真正实现绘制的地方是 ``` drawBackgroundAndRipples(canvas);```

    private void drawBackgroundAndRipples(Canvas canvas) {
        final RippleForeground active = mRipple;
        final RippleBackground background = mBackground;
        final int count = mExitingRipplesCount;
        if (active == null && count <= 0 && (background == null || !background.isVisible())) {
            // Move along, nothing to draw here.
            return;
        }

        final float x = mHotspotBounds.exactCenterX();
        final float y = mHotspotBounds.exactCenterY();
        canvas.translate(x, y);

        final Paint p = getRipplePaint();

        if (background != null && background.isVisible()) {
            background.draw(canvas, p);
        }

        if (count > 0) {
            final RippleForeground[] ripples = mExitingRipples;
            for (int i = 0; i < count; i++) {
                ripples[i].draw(canvas, p);
            }
        }

        if (active != null) {
            active.draw(canvas, p);
        }

        canvas.translate(-x, -y);
    }
 
 
 上面的代码，有一个RippleBackground和一个RippleForeground，RippleBackground是用于绘制Ripple的背景的，单击控件时，我们看到的波纹动画由RippleForeground绘制。调用getRipplePaint()获取到画波纹的画笔，如果count > 0，就调用ripples[i].draw()方法。count表示波纹的计数，1表示一次波纹，2表示2次波纹，最多10次波纹，这里count = 1。
 
 具体实现波纹效果的，其实就是```active.darw(canvas, p)```
 
     public void draw(Canvas c, Paint p) {
        final boolean hasDisplayListCanvas = !mForceSoftware && c instanceof DisplayListCanvas;

        pruneSwFinished();
        if (hasDisplayListCanvas) {
            final DisplayListCanvas hw = (DisplayListCanvas) c;
            drawHardware(hw, p);
        } else {
            drawSoftware(c, p);
        }
    }
 
最终的实现其实就是画圆：

    private void drawSoftware(Canvas c, Paint p) {
        final int origAlpha = p.getAlpha();
        final int alpha = (int) (origAlpha * mOpacity + 0.5f);
        final float radius = getCurrentRadius();
        if (alpha > 0 && radius > 0) {
            final float x = gtCurrentX();
            final float y = getCurrentY();
            p.setAlpha(alpha);
            c.drawCircle(x, y, radius, p);
            p.setAlpha(origAlpha);
        }
    }
 
整体的流程可以看下面的时序图：
 
![image](http://pduvi7ojx.bkt.clouddn.com//18-8-30/35967265.jpg)

*参考：https://blog.csdn.net/ttjjjpf/article/details/43644785*


<h3 id="3">转场动画</h3>

5.0之前的转场动画是通过 Activity 的 overridePendingTransition() 和 FragmentTransaction 的 setCustomAnimation() 实现的，但是只能整个视图一起动画变换。

transition可以做的工作：
* 在开始场景和结束场景中捕捉每一个view的状态
* 根据视图一个场景移动到另一个场景的差异创建一个Animator

共享元素其实就是一个更细化的转场动画，这些动画能让页面过渡更自然。

**猪队友中的实践：**
* 首页推荐、关于页面跳转播放页面
* 侧边栏跳转个人主页
* 个人主页的视频列表跳转播放页面

具体的使用请参考（很详细）： https://github.com/OCNYang/Android-Animation-Set/tree/master/transition-animation

<h4 id="3.1">实现原理</h4>

**先概述下整体的实现流程：**
1. Activity A调用startActivity()， Activity B被创建，测量，同时初始化为半透明的窗口和透明的背景颜色。
2. framework重新分配每个共享元素在B中的位置与大小，使其跟A中一模一样。之后，B的进入变换（enter transition）捕获到共享元素在B中的初始状态。
3. framework重新分配每个共享元素在B中的位置与大小，使其跟B中的最终状态一致。之后，B的进入变换（enter transition）捕获到共享元素在B中的结束状态。
4. B的进入变换（enter transition）比较共享元素的初始和结束状态，同时基于前后状态的区别创建一个Animator(属性动画对象)。
5. framework 命令A隐藏其共享元素，动画开始运行。随着动画的进行，framework 逐渐将B的activity窗口显示出来，当动画完成，B的窗口才完全可见。

![image](http://pduvi7ojx.bkt.clouddn.com//18-8-30/60306741.jpg)

从这个流程中可以看到，共享元素变换并不是真正实现了两个activity或者Fragment之间元素的共享，实际上我们看到的几乎所有变换效果中（不管是B进入还是B返回A）,共享元素都是在B中绘制出来的。

Framework没有真正试图将A中的某个元素传递给B，而是采用了不同的方法来达到相同的视觉效果。A传递给B的是共享元素的状态信息。B利用这些信息来初始化共享View元素，让它们的位置、大小、外观与在A中的时候完全一致。当变换开始的时候，B中除了共享元素之外，所有的其他元素都是不可见的。

随着动画的进行，framework 逐渐将B的activity窗口显示出来，当动画完成，B的窗口才完全可见

**一个重要的元素： Overlay**

共享元素默认其实是绘制在整个view树结构的最上层，在一个叫ViewOverlay的东西上面。你可能没听说过ViewOverlay，他是4.3之后才有的一个新类，它是view的最上面的一个透明的层，添加到ViewOverlay上面的Drawable和view可以被绘制到任何东西的上面，甚至是ViewGroup的子元素

参考： [ViewOverlay与animation介绍 ](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0130/2384.html) 

梳理下了流程后，再撸下源码：

    //使用
    Pair<View, String> pair1 = new Pair<View, String>(holder.mImageView, mContext.getString(R.string.share_element_imageview));
    Pair<View, String> pair2 = new Pair<View, String>(holder.mHeader, mContext.getString(R.string.share_element_header));
    Pair<View, String> pair3 = new Pair<View, String>(holder.mTextView, mContext.getString(R.string.share_element_tv_info));
    ActivityOptionsCompat activityOptionsCompat = ActivityOptionsCompat.makeSceneTransitionAnimation(((Activity) mContext), pair1, pair2, pair3);
    

makeSceneTransitionAnimation的最终实现实在ActivityOptions里面：

![image](http://pduvi7ojx.bkt.clouddn.com//18-8-30/68686023.jpg)

这边设置了mAnimationType为ANIM_SCENE_TRANSITION，并且初始了前一个页面的share view状态。

    @SafeVarargs
    public static ActivityOptions startSharedElementAnimation(Window window,
            Pair<View, String>... sharedElements) {
        ActivityOptions opts = new ActivityOptions();
        final View decorView = window.getDecorView();
        if (decorView == null) {
            return opts;
        }
        final ExitTransitionCoordinator exit =
                makeSceneTransitionAnimation(null, window, opts, null, sharedElements);
        if (exit != null) {
            HideWindowListener listener = new HideWindowListener(window, exit);
            exit.setHideSharedElementsCallback(listener);
            exit.startExit();
        }
        return opts;
    }
    
再次回到开始调用的地方，makeSceneTransitionAnimation执行完成后，就会开始执行exit

    public void startExit() {
        if (!mIsExitStarted) {
            backgroundAnimatorComplete();
            mIsExitStarted = true;
            pauseInput();
            ViewGroup decorView = getDecor();
            if (decorView != null) {
                decorView.suppressLayout(true);
            }
            moveSharedElementsToOverlay();
            startTransition(new Runnable() {
                @Override
                public void run() {
                    if (mActivity != null) {
                        beginTransitions();
                    } else {
                        startExitTransition();
                    }
                }
            });
        }
    }

上面的代码中，可以看到一个熟悉的对象 ```moveShareElementsToOverlay()```，方法已经无法查看了，但是从命名可以猜测它就是将前一个页面上的share view添加到Overlay上面的

    private void startExitTransition() {
        Transition transition = getExitTransition();
        ViewGroup decorView = getDecor();
        if (transition != null && decorView != null && mTransitioningViews != null) {
            setTransitioningViewsVisiblity(View.VISIBLE, false);
            TransitionManager.beginDelayedTransition(decorView, transition);
            setTransitioningViewsVisiblity(View.INVISIBLE, false);
            decorView.invalidate();
        } else {
            transitionStarted();
        }
    }
    
再看最后的执行，其实和上面分析的类似，就是两个窗口的可见状态的变更。

<h4 id="3.2">注意点</h4>

1. 设置到网络数据加载，需要延迟执行共享动画
    
    ![image](http://pduvi7ojx.bkt.clouddn.com//18-8-30/70348934.jpg)

2. 有多个view需要页面共享，需要使用ActivityOptionsCompat.makeSceneTransitionAnimation包装

    ![image](http://pduvi7ojx.bkt.clouddn.com//18-8-30/14732831.jpg)