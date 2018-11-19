## 第二部分 - 属性动画源码解读篇

> 俗话说的好：带着问题看源码，比较不容易迷路！哈哈。

### 前言

**属性动画** 大家应该都很熟悉，基础用法如下：
```
ObjectAnimator animator = ObjectAnimator.ofFloat(view, "rotation", 0, 180);  
animator.setDuration(1000);  
animator.start();
```
代码非常简洁易懂，创建一个ObjectAnimator，实现对view对象做一个0-180度持续1000ms的旋转动画，但细心的同学可能会有这样的疑问：ofFloat方法第二个参数 propertyName 表示方法名，它除了 **"rotation"** 还可以支持其他哪些字符？

接下来我们一起略读一下 Animator 的源码，了解 **propertyName** 字段实现原理的同时，顺便把动画执行的过程也了解一番。

### 源码分析

#### 1. ObjectAnimator 类

首先看到无论是ofFloat、ofInt还是其他，最都会通过 **ObjectAnimator(target, propertyName)** 构建一个ObjectAnimator对象，在ObjectAnimator的构造方法中便会调用下面的setProperty方法来设置一个全局变量 **propertyName**。

```
public void setPropertyName(@NonNull String propertyName) {
    ...
    mPropertyName = propertyName;
    // New property/values/target should cause re-initialization prior to starting
    mInitialized = false;
}
```
下面调用的是设置属性值0-180度的方法：

```
@Override
public void setFloatValues(float... values) {
    if (mValues == null || mValues.length == 0) {
        // No values yet - this animator is being constructed piecemeal. Init the values with
        // whatever the current propertyName is
        ...
        setValues(PropertyValuesHolder.ofFloat(mPropertyName, values));
    } else {
        super.setFloatValues(values);
    }
}
```

#### 2. PropertyValuesHolder 类

上面的源码中看到一个动画中非常重要的类 **PropertyValuesHolder**，它保存了动画过程中所需要操作的属性和对应的值，来看看创建实例的函数：

```
public static PropertyValuesHolder ofFloat(String propertyName, float... values)  
public static PropertyValuesHolder ofInt(String propertyName, int... values)   
public static PropertyValuesHolder ofObject(String propertyName, TypeEvaluator evaluator, Object... values)  
public static PropertyValuesHolder ofKeyframe(String propertyName, Keyframe... values)  
```
其中：
- **propertyName：** 表示 ObjectAnimator 需要操作的属性名。如上面的 "rotation"；
- **values：** 属性所对应的参数（rotation对应角度，alpha对应透明度，甚至可以是一个属性的对象），同样是可变长参数，可以指定多个；
- **TypeEvaluator：** 估值器（*又见面了*），当value是一个自定义的属性对象时，估值器用于控制这个属性对象的变化规律；
- **Keyframe：** 关键帧，value的值最终都会 *均匀的* 转化成对应的Keyframe，这里也支持直接设置关键帧；

很好奇的看了下ofFloat中转换成Keyframe的方法：

```
public static KeyframeSet ofFloat(float... values) {
    boolean badValue = false;
    int numKeyframes = values.length;
    FloatKeyframe keyframes[] = new FloatKeyframe[Math.max(numKeyframes,2)];
    if (numKeyframes == 1) {
        keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f);
        keyframes[1] = (FloatKeyframe) Keyframe.ofFloat(1f, values[0]);
        if (Float.isNaN(values[0])) {
            badValue = true;
        }
    } else {
        keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f, values[0]);
        for (int i = 1; i < numKeyframes; ++i) {
            keyframes[i] = (FloatKeyframe) Keyframe.ofFloat((float) i / (numKeyframes - 1), values[i]);
            if (Float.isNaN(values[i])) {
                badValue = true;
            }
        }
    }
    ...
    return new FloatKeyframeSet(keyframes);
}
```
**ofFloat()** 只设置一个value时默认start是0f，当设置了多个value时，关键帧是均匀分布的:

```
// 3个value时
Keyframe.ofFloat(0f, values[0])
Keyframe.ofFloat(0.5f, values[1])
Keyframe.ofFloat(1f, values[2])
```
要注意这里的 **0.5f** 其实是差值器 getInterpolation()（*又见面了*）的返回值。回头看PropertyValuesHolder提供的ofKeyframe()方法，其实就是为了方便我们控制动画的一些中间效果（如回弹、变速等）。

#### 3. 对 ObjectAnimator 的一个小优化

了解 **PropertyValuesHolder** 类之后，下面这段代码就很好理解了：

```
Keyframe keyframe1 = Keyframe.ofFloat(0.0f, 0);
Keyframe keyframe2 = Keyframe.ofFloat(0.25f, -30);
Keyframe keyframe5 = Keyframe.ofFloat(1.0f, 0);
PropertyValuesHolder rotation = PropertyValuesHolder.ofKeyframe("rotation", keyframe1, keyframe2, keyframe3);
        
PropertyValuesHolder scaleX = PropertyValuesHolder.ofFloat("scaleX", 1.0f, 0.2f);
PropertyValuesHolder scaleY = PropertyValuesHolder.ofFloat("scaleY", 1.0f, 0.2f);
...
ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, rotation, scaleX, scaleY, ...);
animator.setDuration(1000).start();
```
原本需要多个 ObjectAnimator 来完成的动画，通过 PropertyValuesHolder 只需要一个 ObjectAnimator 就可以完成，这样的好处就是减少了 ObjectAnimator 对象的创建，以及减小了部分动画执行的性能消耗。

#### 4. 开始播放动画

在设置完所有属性之后，调用ObjectAnimator.start()方法开始播放动画：

```
private void start(boolean playBackwards) {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    mReversing = playBackwards;
    // Special case: reversing from seek-to-0 should act as if not seeked 
    ...
    mStarted = true;    // 对应isStarted()
    mPaused = false;    // 对应isPaused()
    mRunning = false;   // 对应isRunning()
    mAnimationEndRequested = false; // 动画结束handler释放完成标志
    ...
    AnimationHandler animationHandler = AnimationHandler.getInstance();
    // delay处理
    animationHandler.addAnimationFrameCallback(this, (long) (mStartDelay * sDurationScale));

    if (mStartDelay == 0 || mSeekFraction >= 0) {
        // 正式开始播放
        startAnimation();
        ...
    }
}

/**
 * Called internally to start an animation by adding it to the active animations list. Must be
 * called on the UI thread.
 */
private void startAnimation() {
    ....
    initAnimation();
    mRunning = true; // isRunning() 标识
    if (mSeekFraction >= 0) {
        mOverallFraction = mSeekFraction;
    } else {
        mOverallFraction = 0f;
    }
    if (mListeners != null) {
        notifyStartListeners();
    }
}
```
**start()** 方法有个注意点，对于 startDelay() 的处理，在delay过程中，动画处于 **{isStarted=true,isRunning= false}** 状态，当delay完成执行startAnimation()后动画开始播放，isRunning才会为true；


```
@CallSuper
@Override
void initAnimation() {
    if (!mInitialized) {
        // mValueType may change due to setter/getter setup; do this before calling super.init(),
        // which uses mValueType to set up the default type evaluator.
        final Object target = getTarget();
        if (target != null) {
            final int numValues = mValues.length;
            for (int i = 0; i < numValues; ++i) {
                mValues[i].setupSetterAndGetter(target);
            }
        }
        super.initAnimation();
    }
}
```
**mValues** 就是上面创建的 PropertyValuesHolder，**setupSetterAndGetter()** 设置了这个动画执行的核心方法，属性动画的核心其实是对获取某个属性并对它进行不断的修改：

```
void setupSetter(Class targetClass) {
    Class<?> propertyType = mConverter == null ? mValueType : mConverter.getTargetType();
    mSetter = setupSetterOrGetter(targetClass, sSetterPropertyMap, "set", propertyType);
}

private void setupGetter(Class targetClass) {
    mGetter = setupSetterOrGetter(targetClass, sGetterPropertyMap, "get", null);
}
```
其中：
- **targetClass：** 就是我们最开始设置的执行动画的对象 view.getClass()；
- **sGetterPropertyMap：** 缓存设置过的Method；
- **"set/get":** 用于拼接方法名；

后面的源码量比较多，就不全部贴出来了，一句话就可以概括，根据propertyName（设置首字母大写）拼接"set/get"，通过反射找到对应的方法并缓存到Map中以方便下次使用，至此开头提出的问题已得到答案，并且也知道了动画是如何开始的，下面还有一个关键环节，动画是如何刷新及播放的？

### 动画刷新过程源码

#### 1. AnimationHandler 类

前面的源码中常可以到 **AnimationHandler** 这是一个单例的动画执行管理类，内部维护了一个 **Choreographer** 接收显示系统的时间脉冲(垂直同步信号-VSync信号)，通过Choreographer#FrameCallback与Choreographer进行交互，他们的作用就是确保动画值的设置将发生在相同的线程，以保证动画的同步进行。
```
public final static ThreadLocal<AnimationHandler> sAnimatorHandler = new ThreadLocal<>();
public static AnimationHandler getInstance() {
    if (sAnimatorHandler.get() == null) {
        sAnimatorHandler.set(new AnimationHandler());
    }
    return sAnimatorHandler.get();
}

// Choreographer创建
final Choreographer mChoreographer = Choreographer.getInstance();

// Choreographer#FrameCallback
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        getProvider().postCommitCallback(new Runnable() {
            @Override
            public void run() { //当前帧开始执行
                // do anim
                doAnimationFrame()
                ...
            }
        });
        ...
        getProvider().postFrameCallback(this); //还有动画未完成，设置下一帧的回调，否则不再监听
    }
};
```

#### 2. Choreographer 类

上面的代码是 **AnimationHandler** 中核心的几个类的初始化，其中 **Choreographer** 尤为重要

```
private Choreographer(Looper looper) {
    mLooper = looper;
    mHandler = new FrameHandler(looper);
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;

    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```
这里做了几个初始化操作，根据Looper对象生成，Looper和线程是一对一的关系，每个线程对应一个Choreographer。
1. 初始化FrameHandler。接收处理消息。
2. 初始化FrameDisplayEventReceiver。FrameDisplayEventReceiver用来接收垂直同步脉冲，就是VSync信号，VSync信号是一个时间脉冲，一般为60HZ，用来控制系统同步操作。
3. 初始化mLastFrameTimeNanos(标记上一个frame的渲染时间)以及mFrameIntervalNanos(帧率,fps，一般手机上为1s/60)。
4. 初始化CallbackQueue，callback队列，将在下一帧开始渲染时回调。

```
@Override
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    // 接收VSync信号
    ...
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}

private final class FrameHandler extends Handler {

    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME:
                //回到AnimationHandler
                doFrame(System.nanoTime(), 0);
                break;
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK:
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}
```
接收底层的VSync信号开始处理UI过程。VSync信号由SurfaceFlinger实现并定时发送。FrameDisplayEventReceiver收到信号后，调用onVsync方法组织消息发送到当前线程处理，也就是上面FrameCallback中的doFrame()回调了。通过图表来看一下Choreographer的工作流：
<img src="https://github.com/Chensigou/resource/blob/master/1688934-9324b8eb156bdc5c.png?raw=true" height="320" />

这里还有一个知识点，关于掉帧的处理，第二个信号到来时，Draw操作没有按时完成，导致第三个时钟周期内显示的还是第一帧的内容。

#### 3. AnimationFrameCallback 类

底层VSync信号的处理这里就不深入了，接下来的部分是关于同步信号具体如何通知动画刷新的，AnimationFrameCallback提供了两个回调

```
/**
 * Callbacks that receives notifications for animation timing and frame commit timing.
 */
interface AnimationFrameCallback {
    // 执行动画 Run animation based on the frame time.
    void doAnimationFrame(long frameTime);
    // 每一帧结束调整相关参数
    void commitAnimationFrame(long frameTime);
}
```
通过doAnimationFrame()回调中依次执行了下面的方法

```
public final void doAnimationFrame(long frameTime) {
    ...
    final long currentTime = Math.max(frameTime, mStartTime);
    boolean finished = animateBasedOnTime(currentTime);

    if (finished) {
        endAnimation();
    }
}

boolean animateBasedOnTime(long currentTime) {
    boolean done = false;
    if (mRunning) {
        ...
        float currentIterationFraction = getCurrentIterationFraction(mOverallFraction);
        animateValue(currentIterationFraction);
    }
    return done;
}

@CallSuper
@Override
void animateValue(float fraction) { 
    // 父类ValueAnimator通过差值器估值器计算属性值: mValues[i].calculateValue(fraction);
    super.animateValue(fraction);
    ...
    // 子类ObjectAnimato更新各个动画属性值
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        // 在这里最终为对象设置了属性
        mValues[i].setAnimatedValue(target);
    }
}
```
最终执行的是PropertyValuesHolder类的setAnimatedValue()，通过前面反射得到的方法来设置对象的属性：**mSetter.invoke(target, mTmpValueArray)**，这样就完成了对象属性的更新，再通过一帧帧的刷新完成了动画的展示。

### 结论

绕了一大圈，中途强行插入学习了一波 **PropertyValuesHolder** 和 **Keyframe** 原理和小技巧，还偶遇 **差值器 & 估值器** 了解了它们是如何将动画的工作完整串联起来的，最后还通过 **Choreographer** 了解了系统的时间脉冲...

最后差点忘了解答开头的疑问了，**“ofFloat方法第二个参数 propertyName 除了 "rotation" 还可以支持哪些参数”** ？

**答案就是：** 动画的目标对象‘view’类中，任何一个属性X，只要同时有setX()，getX()一对方法都可以支持，**propertyName字符串，就是"X"**。比如TextView类中，可以有getTextSize()、setTextSize()、getTextScaleX()、setTextScaleX()、getBackgroundColor()、setBackgroundColor()... 还有基类View中，可以有getRotation()、setRotation()、getRotation()、setRotation()...


### 参考
- [Android ObjectAnimator源码分析](https://www.jianshu.com/p/b0e7c92ecb43)
- [Android Choreographer 源码分析](https://www.jianshu.com/p/996bca12eb1d)
- [线程内部的数据存储类 --- ThreadLocal工作原理](https://blog.csdn.net/singwhatiwanna/article/details/48350919)

---

<img src="https://raw.githubusercontent.com/Chensigou/resource/master/logo.png" height="120" />

