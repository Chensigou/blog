### 导读
如今 **App** 中的交互动画越来越丰富，好看好玩的动画总能让人眼前一亮和心情舒畅。但是当产品和设计拿着一些眼花缭乱的动画原型来到开发面前时，很多时候是我们是一脸懵逼和手足无措的。

直播产品中动画的应用场景非常丰富，大到上天入地宇宙爆炸的超级礼物动画，还有冒冒冒个不停的热度机飘心动画，小到触摸反馈的回弹动效，还有各种组合效果。本篇文章前半部分将平时用到知识点做一个整理和导读，后半部分会分享一些项目中的小组件并讲一下设计实现思路，希望对大家喜欢。

## 第一部分 - 基础

> 不知道写动画的文档会不会和写动画一样有趣呢？哈哈。

### 四种实现方式

首先，作为热身，先回忆一下 **Android** 项目中实现动画的一些基本方式

- #### 自定义View
    ```
    // 绘制实现：
    Paint paint = new Paint(); //创建画笔，注一定要放在onDraw外部防止重复创建
    @Override
    protected void onDraw(Canvas canvas) {  
        super.onDraw(canvas);
        canvas.drawCircle(cx, 300, 200, paint); // 绘制一个圆，还可以drawBitmap、drawRect、drawPath...
        ...
    }
    // 动画实现：通过不断改变cx的值，并调用invalidate()方法重绘，最终实现动画；
    
    // 评价：draw()是最底层的绘制实现，View所有的绘制都是通过draw()来实现的，缺点就是实现成本较大；
    ```
    
- #### 逐帧动画
    实现方式：**<animation-list>**（**res/anim/xxx.xml**）或 **AnimationDrawable**类；
    
    评价：优点是可以展现较复杂的动画且实现简单，缺点内存和本地资源空间都占用巨大。

- #### 补间动画
    应用场景：平移缩放旋转透明组合效果，页面进出场动效、View基础动效；

    实现方式：**<translate/alpha/set...>**（**res/anim/xxx.xml**）或 **Animation**相关类；
    
    进阶：**delay、setRepeatCount、addListener、setInterpolator...**
    
    评价：优点是使用方便，缺点是++仅产生效果正式属性不会改变++。
    
- #### 属性动画
    介绍：属性动画是 **Android 3.0** 加入的动画模式，它的出现似乎就是为了弥补补间动画无法改变View原本属性的缺陷，同时还拓展了作用对象（可以对属性进行动画）以及动画的效果（不仅是4种基础效果，所有属性都可以实现）；

    **ValueAnimator**类：通过 **addUpdateListener**() 监听控制值的变化，再手动给对象设置属性，从而实现动画效果；
    
    **ObjectAnimator**类：继承自**ValueAnimator**类，内部自动给对象设置属性，并调用View的 **invalidate**() 方法重绘View实现动画效果；
    
    **AnimatorSet**类：可以实现组合动画，他们三个都继承自 **android.animation.Animator** 类；
    
    评价：属性动画配合自定义View的绘制，稍微发挥一下想象力，就可以实现你想要的动效了，接下来要注意的就是++性能和效率问题++了。
    
- #### 其他动画
    第三方的开源库，比如：Svga，Lottie，还有后面要讲的Lwf等...他们的优势是可以实现较复杂的动画，性能较好，缺点是动画过程中不方便做太多交互，且Cpu和内存的占用较高。

### 差值器 & 估值器
差值器和估值器是实现复杂动画效果的关键，换一种说法，通过使用差值器和估值器，我们可以：简化动画过程，优化动画效果，让动效更加饱满生动。

- #### 差值器
    话不多说，差值器就如其名一样，神秘而令人向往，直接看下demo：

    <img src="https://raw.githubusercontent.com/Chensigou/resource/master/944365-0a9435149480729e.gif" width="320" />
    
    原本 **TextView** 是由 **A** 匀速运动到 **B** 的，在 **translate** 动画加上了差值器之后，我们便实现了很多复杂的运动轨迹（**scale、rotation、alpha** 加上差值器也会有全新的动画轨迹），这里挑选第一个先加速再减速的差值器中看一下源码的实现：
    ```
    // AccelerateDecelerateInterpolator 差值器：先加速再减速
    public class AccelerateDecelerateInterpolator implements Interpolator, NativeInterpolatorFactory {  
        ...
        public float getInterpolation(float input) { 
            // input的取值范围是0到1，是一个动画完整过程。
            return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f; //ADI公式
        }  
    }
    ```
    仔细看代码就会有一个疑问，源码中的ADI公式这个到底是计算了什么呢，推荐一个很好用的[差值器工具](http://inloop.github.io/interpolator/)，我们将上面的ADI公式输入右侧公式栏，可以得到下面的曲线：
    
    <img src="https://github.com/Chensigou/resource/blob/master/WX20180810-193816@2x.png?raw=true" width="500" />
    
    任何一段曲线都可以找到一个线性公式，通过公式实现 **Interpolator** 接口便可以实现一个对应的差值器。上面工具右下角的 **Library** 中还**提供了一些设计工具中常用的差值器公式，可供我们自定义使用**，比如下面这设计常用（如Principle/Sketch等设计工具中自带）的 **Spring差值器** 的实现方式：
    
    ```
    // 根据工具获得 Spring公式：
    // factor = 0.4
    // pow(2, -10 * x) * sin((x - factor / 4) * (2 * PI) / factor) + 1
    
    /**
     * 根据获得的公式 自定义SpringInterpolator
     */
    class SpringInterpolator : Interpolator {
        private var factor = 0.4f // 默认值
    
        constructor(factor: Float) {
            // 视觉定的默认值
            this.factor = factor
        }
    
        override fun getInterpolation(input: Float): Float =
                (Math.pow(2.0, -10.0 * input) * Math.sin((input - factor / 4) * (2 * Math.PI) / factor) + 1).toFloat()
    }
    ```
    当然，如果交互设计师突然设计灵感爆发，全新设计出一套动效属性（差值器），这时候我们该如何实现呢，接下来介绍一款 [三次方曲线差值器工具](http://cubic-bezier.com/#.17,.67,.83,.67)，实现原理简单描述下：
    ```math
    B(t) = (1-t)^3P1+3(1-t)^2tP1+3(1-t)t^2P2+t^3P3,t∈[0,1]
    ```
    基于贝塞尔曲线三阶方程，通过起点P0(0,0)，随机点P1，随机点P2，终点P3(1,1)，确定四个点即可完成一条曲线。通过这款工具画出想要的曲线后，接下来要做的就是简化公式并在自定义的差值器中实现，具体不做详述[参考地址](https://blog.csdn.net/xsl_bj/article/details/47722489)，到这里差值器部分就完成了，**实际应用的效果可以关注下面的"爪子动画"。**

- #### 估值器
    插值器 **Interpolator** 决定了值的变化规律（匀速、加速、变速... 即变化趋势），而接下来的具体变化数值则是由估值器 **TypeEvaluator** 决定的。

    ```
    PointF startValue = new PointF(rootView.getWidth()/2, 0); //起点
    PointF endValue = new PointF(rootView.getHeight()/2, 0);  //终点
    new ValueAnimator().setObjectValues(startValue, endValue);//初始化
    
    // 实现TypeEvaluator估值器接口
    public class ObjectEvaluator implements TypeEvaluator{  
        // 在evaluate（）里写入对象动画过渡的逻辑
        @Override  
        public Object evaluate(float fraction, Object startValue, Object endValue) {  
        // evaluate() 方法里处理对象动画过渡的逻辑
        // 参数说明
        // fraction：根据它来计算当前动画的值（插值器getInterpolation（）的返回值）
        
        // startValue、endValue：上面初始化时设置，动画的初始值和结束值
        value = fraction * ... // 写入对象动画过渡的逻辑，计算运动轨迹，必须

        return value; // 返回对象动画过渡的逻辑计算后的值
    } 
    ```
    看代码其实就很清晰了，定义好起点终点交给估值器，两点之间以什么轨迹运动便由TypeEvaluator统一封装控制。顺带说一句，startValue的类型是Object，它可以代表一个坐标、一个属性对象，又或是一个字符串，任何对象的属性都可以哦。
    
    有兴趣的同学快去实践一下吧（[摩拳擦掌~](https://blog.csdn.net/IO_Field/article/details/53101452)）

### 一些属性动画的坑

- startDelay方法注意点（isStarted & isRunning）：
    
    通过startDelay创建动画，创建后isStarted状态为true，当动画真正start的时候isRunning状态为true；（参考 [属性动画源码略读](https://git.corp.plu.cn/developershare_app/document_tech/blob/master/%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%E7%AF%87.md)）；

- 如何减少ObjectAnimator对象的创建：
    
    通过ObjectAnimator.ofPropertyValuesHolder(view, scaleX, scaleY, ...)的方式创建动画，可以适当的减少不必要的对象创建，优化流畅度。（参考 [属性动画源码略读](https://git.corp.plu.cn/developershare_app/document_tech/blob/master/%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%E7%AF%87.md) --- 小优化部分）；

- 动画到底是在哪个线程执行：

    动画其实可以在任何线程执行，唯一的要求就是必须要有Looper（原生的Thread就不行，必须要使用HandlerThread），如果动画中属性的变换需要有大量的计算，才建议在子线程startAnim()，一般的动画在主线程执行即可。

- 内存泄露问题：

    动画执行完成或者动画终止后，内部的handler会自动释放，但如果动画设置了setRepeatMode(ValueAnimator.REVERSE) 循环播放，必须在View关闭时手动结束动画来释放动画资源；
    
- 多层级ViewGroup动画效率：
    
    尽量针对根部的View做动画，一个View本身绘制需要多少时间，对这个View做动画就是不断的绘制他本身。（对父View执行动画时，会导致所有的子View都发生重绘，这样对性能的影响十分严重，在做动画时层级的问题一定要重视）

- 硬件加速的使用：

    [通过Hardware Layer提升Android动画性能](https://www.jianshu.com/p/f1feafffc365)：使用硬件加速后，GPU会把View的渲染加到缓冲区来复用，可以很大程度上降低绘制的消耗，不过这个方法的使用条件很苛刻，仅针对属性动画，且加速的View对象在动画过程中不能有其他的重绘发生，否则反而会带来很大的额外消耗。看一下属性设置（如：setScale() ）的部分源码：
    ```
    void invalidateViewProperty(boolean invalidateParent, boolean forceRedraw) {
        if (!isHardwareAccelerated()
                || !mRenderNode.isValid()
                || (mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
            if (invalidateParent) {
                invalidateParentCaches();
            }
            if (forceRedraw) {
                mPrivateFlags |= PFLAG_DRAWN; // force another invalidation with the new orientation
            }
            invalidate(false);
        } else {
            damageInParent();
        }
        if (isHardwareAccelerated() && invalidateParent && getZ() != 0) {
            damageShadowReceiver();
        }
    }
    ```
    可以看到 **isHardwareAccelerated** 字段，如果未开启硬件加速，还是通过invalidate()来通知View重绘，开始硬件加速后，就通过 damageInParent() 来绘制了，具体的效果大家可以亲自试一下。

### 参考
- [给高级 Android 工程师的进阶手册（自定义View1-8章）](http://hencoder.com/ui-1-1/)
- [Android 动画：这是一份详细 & 清晰的动画学习指南](https://juejin.im/post/5aea7063f265da0b9b072758)
- [一篇很详细的 属性动画 总结](https://blog.csdn.net/carson_ho/article/details/72909894)
- [差值器一（内附工具）](http://www.sohu.com/a/192971808_659256)
- [差值器二（内附工具）](https://blog.csdn.net/xsl_bj/article/details/47722489)
- [通过Hardware Layer提升Android动画性能](https://www.jianshu.com/p/f1feafffc365)


---

<img src="https://raw.githubusercontent.com/Chensigou/resource/master/logo.png" height="120" />