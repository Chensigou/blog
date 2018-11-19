## 第三部分 - 爪子动画示例剖析

> 其实作为开发，能够得到一个优秀的交互动画的开发机会，也是很难得的，一定要好好珍惜，哈哈。

### 概述

赠送道具作为一个核心的营收过程，交互体验的重要性不言而喻。

<img src="https://github.com/Chensigou/resource/blob/master/video2gif_20180827_103501.gif?raw=true" height="320" />

这一版的爪子设计还是很不错的，有不少细节很用心。要实现这个控件，首先做的是根据实际情况拆分子控件，下面就开始具体的实现部分。

#### 1. PressEffectView 小爪子部分
这是一个自定义TextView（可以设置文字，如520，1314等），具有按压的反馈动效及回弹动效，所以首先要做的就是对触摸事件的监听：

```
override fun onTouch(v: View?, event: MotionEvent?): Boolean {
    when (event?.action) {
        MotionEvent.ACTION_DOWN -> {
            if (scaleAnim?.isRunning == true) {
                scaleAnim?.cancel()
            }
            if (animator?.isStarted != true) {
                animator?.start()
            }
        }
        MotionEvent.ACTION_OUTSIDE,
        MotionEvent.ACTION_CANCEL,
        MotionEvent.ACTION_UP -> {
            animator?.cancel()
            //长按触发回弹特效
            val xAnim = PropertyValuesHolder.ofFloat("scaleX", ratio, 1f)
            val yAnim = PropertyValuesHolder.ofFloat("scaleY", ratio, 1f)
            scaleAnim = ObjectAnimator.ofPropertyValuesHolder(this, xAnim, yAnim)
            scaleAnim?.interpolator = SpringInterpolator()
            scaleAnim?.duration = 100 + (ratio * 100).toLong()
            scaleAnim?.start()
        }
    }
    return false
}
```
这里可以看到两个动画的对象

1. **scaleAnim** 这是一个ObjectAnimator，用于展示小爪子放大后的回弹动效，通过 **PropertyValuesHolder** 将scaleX、scaleY属性整合，减少ObjectAnimator对象的创建，同时使用了 **SpringInterpolator** 弹性差值器，提供了回弹效果，[第一章](https://git.corp.plu.cn/developershare_app/document_tech/wikis/Android-%E7%89%B9%E6%95%88%E5%8F%8A%E5%8A%A8%E7%94%BB%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E5%9F%BA%E7%A1%80%E5%9B%9E%E9%A1%BE)里已经做过详细介绍了；

2. **animator** 这是一个ValueAnimator，在ACTION_DOWN时开始执行放大的动画，使用了一个 **DecelerateInterpolator** 减速差值器来提升按压效果，实现很简单下面是代码；

    ```
    // 创建基础
    animator = ValueAnimator.ofFloat(1.08f, 1.5f)
    animator?.interpolator = DecelerateInterpolator()
    animator?.duration = 500
    animator?.addUpdateListener(this@PressEffectView)
    ...
    
    override fun onAnimationUpdate(animation: ValueAnimator?) {
    //  val fraction = animation?.animatedFraction // 动画进度(t/duration)
        ratio = animation?.animatedValue as? Float ?: return // 获取当前值
        scaleX = ratio
        scaleY = scaleX
    }
    ```

#### 2. ProgressEffectView 主连击部分
自定义View，具有按压的反馈及一个圆形的进度Progress（图中的淡黄色边缘就是倒计时的Progress），这里的圆形进度以及内部的圆都是通过canvas绘制的，方法如下：

```
override fun onDraw(canvas: Canvas) {
    // 设置圆心实心背景
    canvas.drawCircle((width / 2).toFloat(), (height / 2).toFloat(), innerRadius, bgPaint)
    if (currentProgress >= 0) {
        // 计算进度条宽度的中心点，注意防止重复创建
        if (oval == null) {
            val r = strokeWidth / 2
            oval = RectF(width / 2 - radius - r, height / 2 - radius - r, width / 2 + radius + r, height / 2 + radius + r)
        }
        // 进度条颜色，第三个参数（正/负）代表旋转方向（顺时针/逆时针）
        ringPaint?.color = ringColor
        canvas.drawArc(oval, -90f, (currentProgress / totalProgress - 1) * 360, false, ringPaint)
    }
}
```
**onDraw()** 方法执行的次数会很多，所有一定要注意，不要在其中创建对象，不要耗时操作，否则会严重影响性能。接下来还是老的套路，重写 **onTouch()** 方法：
```
override fun onTouch(v: View?, event: MotionEvent?): Boolean {
    when (event?.action) {
        MotionEvent.ACTION_DOWN -> {
            ...
            evaluator?.ratio = 0.96f
            evaluator?.isPressing = true
        }
        MotionEvent.ACTION_OUTSIDE,
        MotionEvent.ACTION_CANCEL,
        MotionEvent.ACTION_UP -> {
            ...
            evaluator?.isPressing = false
            if (animator?.isRunning == true) {
                animator?.currentPlayTime = 0
            }
        }
    }
    return false
}
```
**currentPlayTime** 方法用于重新开始播放动画，这里还用到了一个简单的 **evaluator** 估值器:

```
public class ProgressEvaluator implements TypeEvaluator<ProgressValue> {
    private boolean isPressing;
    private float ratio = 1F;

    @Override
    public ProgressValue evaluate(float fraction, ProgressValue startValue, ProgressValue endValue) {
        ProgressValue progressValue = new ProgressValue();
        progressValue.progress = dealProgress(fraction, startValue.progress, endValue.progress);
        progressValue.ratio = dealRatio();
        return progressValue;
    }

    private float dealProgress(float fraction, float startProgress, float endProgress) {
        float difference = endProgress - startProgress;
        return startProgress + difference * fraction;
    }

    private float dealRatio() {
        if (isPressing) {
            if (ratio > 0.8f) {
                ratio -= 0.02f;
            }
            if (ratio < 0.8f) {
                ratio = 0.8f;
            }
        } else {
            if (ratio < 1) {
                ratio += 0.03f;
            }
            if (ratio > 1) {
                ratio = 1f;
            }
        }
        return ratio;
    }
    ...
}
```
使用这个估值器的目的很简单，onDraw()方法中依赖了2个核心变量currentProgress（进度）和innerRadius（半径），估值器可以集中管理动画运行中变量的设置，也不需要创建多个动画导致重复调用invalidate()触发刷新。

```
override fun onAnimationUpdate(animation: ValueAnimator?) {
    val value = (animation?.animatedValue as? ProgressValue?) ?: return // 获取当前值
    currentProgress = value.progress //进度值
    innerRadius = radius * value.ratio //半径值

    ...
    invalidate()
}
```
最后在 **onAnimationUpdate()** 回调中便可以获取到估值器中设置好的变量值，再调用invalidate()通知View重绘即可。

#### 3. BlowUpView 波纹部分
自定义View，根据View在屏幕中的位置绘制波纹动效，看过上面的两个自定义View之后，这个波纹部分其实几乎是一样的套路了，直接看代码吧：
```
fun addBlowUpAnim(view: View?, radius: Int, color: Int) {
    if (lastView != view) {
        view?.getLocationInWindow(location) //获取在当前窗口内的绝对坐标
        this.px = location[0].toFloat() + (view?.width?.toFloat() ?: 0f) / 2
        this.py = location[1].toFloat() + (view?.height?.toFloat() ?: 0f) / 2 - statusBarHeight
        this.lastView = view
    }
    ...

    if (animator?.isStarted == true) {
        animator?.currentPlayTime = 0
    } else {
        animator?.duration = if (isMain) 240 else 320
        animator?.setObjectValues(BlowUpValue(1F, 100), BlowUpValue(if (isMain) 5.8f else 13.8f, 0))
        animator?.setEvaluator(evaluator)
        animator?.start()
    }
}

override fun onDraw(canvas: Canvas) {
    // 设置圆心实心背景
    bgPaint?.alpha = bAlpha
    canvas.drawCircle(px, py, radius * bRatio, bgPaint)
}
```
这里用到了一个获取View在Window中的绝对位置的方法，通过绝对位置找到波纹的中心点，便可以绘制圆形了，剩余的工作很简单就不再做详细的描述了。

#### 4. ClawsComboView 主体部分
全屏大小的Framlayout，是整个控件的容器，在这个容器里将上面说到的自定义View进行组装，还涉及到很多具体的业务功能，比如数据加载、资源释放、出场动画、倒计时展示以及相关事件向上层回调，着重讲一下出场动画吧，就是开头动画中小爪子弹出的动画：

<img src="https://github.com/Chensigou/resource/blob/master/WX20180829-230941@2x.png?raw=true" height="320" />

核心思想就是通过角度计算 **正/余弦值** 得到 **x/y轴坐标**，一共四步：
1. 计算出每个爪子之间的角度值
2. 计算起始角度， startDegree = -(totalDegree - 90) / 2
3. 计算正/余弦值，Math.cos/sin(curDegree)
4. 通过属性动画，将爪子View移动到点 (x, y) 的位置

```
/**
 * 初始化组合礼物，最多取4个，45度对齐（增加自定义送礼按钮后最多5个按钮）
 */
private fun initGroupGifts() {
    if (mGroupGifts == null) return
    if (showAnimatorSet.isRunning) return
    ...
    //设置爪子移动距离
    length = ScreenUtil.dip2px(context, 74f)
    val optionsList = arrayOf("", "66", "199", "1314", "3344")
    val size = optionsList.size
    val totalDegree = when (size) {
        4 -> TOTAL_DEGREE[1]
        3, 1 -> TOTAL_DEGREE[2]
        2 -> TOTAL_DEGREE[3]
        else -> TOTAL_DEGREE[0]
    }
    // 根据按钮总数计算每个按钮的夹角度数
    val degree = if (size <= 1) totalDegree / 2 else totalDegree / (size - 1)
    // 计算初始角度，45度对称
    val startDegree = -(totalDegree - 90) / 2
    for (i in 0 until size) {
        // 角度转弧度，起始角度startDegree度，依次增加degree度
        val curDegree = Math.toRadians(((if (size <= 1) 1 else i) * degree + startDegree).toDouble())
        // 正/余弦计算坐标
        val width = length * Math.cos(curDegree)
        val height = length * Math.sin(curDegree)
        // 设置位置
        val xAnim = PropertyValuesHolder.ofFloat("translationX", 0f, (-width).toFloat())
        val yAnim = PropertyValuesHolder.ofFloat("translationY", 0f, (-height).toFloat())
        val transAnim = ObjectAnimator.ofPropertyValuesHolder(mGroupGifts[i], xAnim, yAnim)
        showAnimatorSet.play(transAnim)
    }
    showAnimatorSet.interpolator = SpringInterpolator(0.6f)
    showAnimatorSet.duration = 360
    showAnimatorSet.start()
}
```
这里同样使用了 **SpringInterpolator** 弹性差值器，通过 **AnimatorSet** 将所有爪子位移动画同时播放，这样爪子的弹出动画就完成了！


### 最后

细节方面的处理，要多体验多思考多讨论，好的体验都是一步步优化出来的！[所有的示例Demo都在这里哦，仅内部人员可以体验！](https://git.corp.plu.cn/AppComponentLibs/LongzhuAnim.git)

---

<img src="https://raw.githubusercontent.com/Chensigou/resource/master/logo.png" height="120" />
