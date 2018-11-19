## 第四部分 - Surface动画示例分析

> 斗胆的说一句，就是那个飘星星的动画，开启了大秀场直播时代，哈哈。

### 概述

最开始做秀场直播的时候，飘星动画几乎是每家直播的标配，这个动画的效果好像直接关系到了秀场直播的体验。由于这个动画伴随着大量对象的创建、绘制和销毁，所以对性能的要求很高，开始我们也尝试过几种方式来实现，在一些高人气房间里性能总有点吃力，最终的方案是通过 **SurfaceView** 这个类来实现动画。

### SurfaceView 介绍

**SurfaceView** 的绘制可以由两个线程控制，多线程的使用大大提升了效率：
1. UI线程，所有SurfaceView和SurfaceHolder.Callback的方法都应该在UI线程里调用；
2. 渲染线程，在SurfaceCreated后创建线程用于更新属性变量以及绘制操作，要注意的地方是在SurfaceDestory之后渲染线程便无法再进行绘制了，需要做好同步的工作。

**SurfaceView** 最核心的特点他有独特的双缓冲机制，这种双缓冲技术需要两个图形缓冲区，前端缓冲区是正在渲染的图形缓冲区，而后端缓冲区是接下来要渲染的图形缓冲区，缓冲区的使用可以大大减低读写消耗，提升渲染效率。

<img src="https://github.com/Chensigou/resource/blob/master/1592280-bea74430778c80b1.png?raw=true" height="270" />

### 飘信心动画

老样子先瞄一眼最终效果：

<img src="https://github.com/Chensigou/resource/blob/master/video2gif_20180831_151117.gif?raw=true" height="280" />

#### 1. Heart 类：单个飘动对象

**Heart** 类中包含了：
1. 动画的运动轨迹矩阵（含位置及旋转信息等）及透明度等信息；
2. 动画的内容资源；
3. 动画的进度数据；
4. 绘制方法，根据提供的Canvas和Paint绘制动画对象；
5. 重置方法，用于复用时重置状态；

```
/**
 * 飘心动画中的对象
 * <p>
 * 注：可以自定义图片，传bitmap即可
 */
public static class Heart {
    public Transform transform; // 动画轨迹属性
    public Bitmap bitmap; // 动画图片
    public long increase; // 动画进度，每绘制一次加1，直至increase*渲染间隔=动画时长

    public Heart(Bitmap bitmap) {
        this.bitmap = bitmap;
        transform = new Transform();
    }

    /**
     * 停止动画，重置属性
     */
    public void reset() {
        increase = 0;
        if (transform != null) {
            transform.reset();
        }
    }

    /**
     * 绘制自己
     */
    public void draw(Canvas canvas, Paint paint, boolean isShaded) {
        if (transform.hasTransform()) {
            if (!bitmap.isRecycled()) {
                ...
                paint.setAlpha(alpha);
                canvas.drawBitmap(bitmap, transform.getMatrix(), paint);
            }
        }
    }
}
```

**HeartPool** 对象池的使用很关键，为了防止 **Heart** 对象的无限创建，必须通过对象池进行复用，另外需要注意的还有 Heart中的 **Bitmap** 对象，同样需要缓存和复用；

```
/**
 * 对象池
 */
public static class HeartPool {
    private WeakHashMap<Integer, Bitmap> bitmapWeakHashMap = new WeakHashMap<>();//位图缓存
    private LinkedList<Heart> cacheHearts = new LinkedList<>();//缓存

    /**
     * 获取随机的图片
     */
    public Bitmap getRandomBitmap(Context context) {
        int resPos = random.nextInt(HEARTS.length);
        Bitmap bitmap = bitmapWeakHashMap.get(resPos);
        if (bitmap == null || bitmap.isRecycled()) {
            bitmap = BitmapFactory.decodeResource(context.getResources(), HEARTS[resPos]);
            bitmapWeakHashMap.put(resPos, bitmap);
        }
        return bitmap;
    }
    
    /**
     * 创建漂浮对象，管理对象池
     */
    public synchronized Heart create(Bitmap bitmap) {
        Heart heart = null;
        if (cacheHearts.size() > 0) {
            try {
                heart = cacheHearts.removeFirst();
                if (heart.bitmap != bitmap) {
                    heart.setBitmap(bitmap);
                }
                heart.reset();
            } catch (Exception e) {
                e.printStackTrace();
            }
        } else {
            heart = new Heart(bitmap);
        }
        return heart;
    }

    /**
     * 回收对象，并加入到缓存池中等待复用
     */
    public void recycleHeart(Heart heart) {
        if (heart == null) {
            return;
        }
        if (!cacheHearts.contains(heart)) {
            cacheHearts.add(heart);
        }
    }

    /**
     * 清理对象池，回收bitmap
     */
    public void clean() {
        cacheHearts.clear();
        if (bitmapWeakHashMap.size() > 0) {
            for (Integer key : bitmapWeakHashMap.keySet()) {
                Bitmap bitmap = bitmapWeakHashMap.get(key);
                BitmapHelper.recycleBitmap(bitmap);
            }
        }
        bitmapWeakHashMap.clear();
    }
}
```

#### 2. AnimTransform 类： 单个动画属性配置类

这个类是单个动画完整的管理类：

```
createPath(View): Path //初始化运行路径
setStartDelay(long): void //设置开始绘制前的间隔时间
isWanting(): boolean //是否在等待执行
draw(Canvas, Paint, boolean): void //绘制自己到Canvas中
getHeart(): Heart //获取对象
transform(): boolean //重新计算位置变换
checkBound(float): boolean //检测边界
recycle(HeartPool): void //回收资源
```

上面接口中核心的方法有两个：
1. **createPath()**，这个方法的作用是使用 **贝塞尔曲线** 创建一个曲线运动轨迹的 **Path** 对象;
2. **transform()**，这个方法的作用和属性动画中的差值器和估值器类似：
3. **checkBound(float)**，这个方法的作用是检测边界，业务上做淡出效果，并判断动画是否结束；

```
/**
 * 创建动画路径，获取飘动的随机位置点
 */
private Path createPath(View view) {
    // TODO: 2016/12/9 路径还可以优化，出来后总是偏右运行
    int x = (int) (mConfig.xRand * (0.75f - Math.random())); // X轴随机位移点
    int d2 = mConfig.animLength * 3 / 2; // Y轴移动距离
    int factor = d2 / mConfig.bezierFactor; // 贝塞尔变换速度
    int y = view.getHeight() - mConfig.initY; // 总高度-初始高度=Y初始点
    int y2 = y - d2; // 第二阶段移动初始点

    Path p = new Path();
    //设置Path的初始点
    p.moveTo(mConfig.initX, y);
    //曲线移动一段距离至点(x, y2)，之后为直线运动
    p.cubicTo(mConfig.initX, y, x, y2 + factor, x, y2);
    return p;
}

/**
 * 动画变换操作
 *
 * @return false：动画完成
 */
public boolean transform() {
    long increase = this.heart.getIncrease();
    // increase * 渲染间隔 = 动画已进行时长
    if (increase > mStartDelay + mConfig.animDuration) {
        // 或动画持续时间已结束
        return false;
    }
    // 动画总进度
    float factor = (float) (increase - mStartDelay) / mConfig.animDuration;
    if (factor < 0) factor = 0; // 边界处理

    //TODO：这一部分其实就是差值器&估值器的工作
    Matrix matrix = heart.getTransform().getMatrix();
    float scale = 1.0f; // view的正常大小值
    float step0 = 0.03f; // 第一阶段总比例2%
    float step1 = 0.12f; // 快速变大阶段总比例24%
    float step2 = 1.0f; // 第三阶段总比例100%
    float distanceFactor = 1.0f; // view正常飘动的速度

    if (factor < step0) {
        // [0, step0)时间区间里，完成[0.0D, 0.25D]的放大变化
        scale = interpolator(factor, 0.0D, step0 + 0.0006D, 0.0D, 0.25D);
        // [0, step0)时间区间里，完成[0.0D, step0]的运动区间
        distanceFactor = interpolator(factor, 0.0D, step0 + 0.0006D, 0.0D, step0);
    } else if (factor < step1) {
        // [step0, step1)时间区间里，完成[0.25D, interpolator]的放大变化
        scale = interpolator(factor, step0 - 0.0006D, step1 + 0.0006D, 0.25D, scale);
        // [step0, step1)时间区间里，完成[step0, step1 + 0.03D]的运动区间
        distanceFactor = interpolator(factor, step0 - 0.0006D, step1 + 0.0006D, step0, step1 + 0.03D);
    } else if (factor <= step2) {
        // [step1, step2]时间区间里，完成[step1 + 0.03D, step2]的运动区间
        distanceFactor = interpolator(factor, step1 - 0.0006D, step2 + 0.0006D, step1 + 0.03D, step2);
    }
    if (mPm != null) {
        // 获取distance上的点配置的matrix位置信息（POSITION_MATRIX_FLAG:位置信息）
        float distance = mPm.getLength() * (distanceFactor);
        mPm.getMatrix(distance, matrix, PathMeasure.POSITION_MATRIX_FLAG);
    }
    matrix.preScale(scale, scale);//接下来再乘这个矩阵,作用是绕着中点缩放
    // 设置view旋转度数
    matrix.postRotate(mRotation * factor);
    //边界检查,防止左边界截断、
    return !checkBound(factor);
}

/**
 * 边缘处理，接近边缘时，加速透明，不可逆
 *
 * @param factor：当前进度
 * @return true：超过边界，完全透明
 */
private boolean checkBound(float factor) {
    float alpha; // view的初始透明度
    Transform transform = heart.getTransform();
    // 当接近边缘距离为width/2时，触发加速透明
    int padding = heart.getBitmap().getWidth() / 4;
    float[] values = new float[9];
    transform.getMatrix().getValues(values);
    if (values[2] < padding) {
        // 触发加速透明
        if (mOverFactor == 1.0f) {
            mOverFactor = factor;
            mAlpha = transform.getAlpha();
        }
    }

    if (mOverFactor != 1.0f) {
        // 加速透明，用总进度10%的时间完成全透明变换
        factor = factor > mOverFactor + 0.1D ? mOverFactor + 0.1f : factor;
        alpha = interpolator(factor, mOverFactor - 0.00006, mOverFactor + 0.10006D, mAlpha, 0.0D);
    } else {
        // 透明度只会越来越小
        alpha = interpolator(factor, 0.0D, 1.00006D, 1.0D, 0.0D);
    }
    // 透明度不可逆
    alpha = Math.min(Math.abs(alpha), heart.getTransform().getAlpha());
    // 设置view的透明度
    heart.getTransform().setAlpha(alpha);
    return alpha == 0;
}
```

#### 3. HeartAnimSurfaceView 类 --- 动画的核心SurfaceView

**HeartAnimSurfaceView**
类就是控制整个动画的创建、渲染等核心部分了，同时还包含了很多业务逻辑，比如说定时/批量添加动画批量添加动画、暂停、恢复、清屏、释放等产品功能，业务的东西不做详述，只讲两个小坑；

- 对象的缓存队列，由于涉及到大量主线程和渲染线程的数据修改操作（比如：遍历渲染对象队列时，主线程同时在往队列中添加新的对象），必须要使用类似 **[CopyOnWriteArrayList](https://www.cnblogs.com/xrq730/p/5020760.html)** 这种线程安全的List来存储对象数据，代价是由于内部使用了锁，效率会有所损失；
    ```
    CopyOnWriteArrayList<AnimTransform> animList = new CopyOnWriteArrayList<>(); //动画列表
    ```

- SurfaceView是纵向Z轴排列的，他的层级和透明度设置也需要手动在代码中设置；
    ```
    // 设置surfaceHolder
    setZOrderMediaOverlay(true);
    surfaceHolder = getHolder();
    surfaceHolder.addCallback(this);
    surfaceHolder.setFormat(PixelFormat.TRANSLUCENT);
    ```

- 渲染需要注意的是，从** lockCavas()** 到 **unlockCanvasAndPost()** 的过程中一定要保持同步。不能线程A调用了lockCavas()，还没有unlock，另一个线程B就跑来lock，这时会直接抛出异常，解决方法是通过同步锁维护一个 **isVisible** 来保证一次绘制的完整执行；
    ```
    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        synchronized (animLock) {
            isVisible = false;
        }
    }
    
    @Override
    protected void onVisibilityChanged(View changedView, int visible) {
        super.onVisibilityChanged(changedView, visible);
        synchronized (animLock) {
            isVisible = visible == VISIBLE;
        }
    }
    
    //渲染线程
    public void run() {
        while (isRun) {
            synchronized (animLock) {
                if (isVisible) {
                    lockCanvas(Rect r) //开始绘制，锁住画布
                    ...
                    canvas.draw()
                    unlockCanvasAndPost(Canvas c) //解锁绘制完成画布，执行渲染操作
                }
            }
        }
    }
    ```

#### 4. RenderThread 类 --- 渲染线程

最后就是渲染线程了，去掉其中各种异常的判断，核心的代码就是循环遍历 **AnimTransform** 更新位置并绘制到canvas中，最后还要手动调用线程的 **sleep()** 方式增加绘制间隔；

```
/**
 * 渲染线程，不断执行heart.draw()绘制星星❤️
 */
private class RenderThread implements Runnable {
    private boolean isRun; // 控制线程开关
    private boolean isRender; // 暂停/开始

    public RenderThread() {
        isRun = true;
        isRender = true;
    }

    /**
     * 关闭绘制线程
     */
    public void stop() {
        isRun = false;
    }

    /**
     * @param isRender：false时暂停绘制
     */
    public void render(boolean isRender) {
        this.isRender = isRender;
    }

    @Override
    public void run() {
        while (isRun) {
            Context context = mContextWf.get();
            if (context == null) {
                isRun = false;
                return;
            }
            if (surfaceHolder == null) {
                isRun = false;
                return;
            }
            Canvas canvas = null;
            long start = System.currentTimeMillis();
            synchronized (animLock) {
                if (isVisible && getWidth() != 0 && getHeight() != 0) { // view不可见、未加载时，不绘制
                    try {
                        if (HeartAnimSurfaceView.this.isRender) {
                            if (unlockCanvas != null) {
                                surfaceHolder.unlockCanvasAndPost(unlockCanvas);
                                unlockCanvas = null;
                            }
                            canvas = surfaceHolder.lockCanvas();
                            if (canvas != null) {
                                canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR); //清屏
                                if (isRender && animList != null) {
                                    for (AnimTransform transform : animList) {
                                        if (transform == null) continue;
                                        // 设置进度增量
                                        transform.getHeart().setIncrease(sleep);
                                        // 一组动画中每个动画间需要有间隔
                                        if (transform.isWanting()) continue;
                                        boolean isTransForming = transform.transform();
                                        if (isTransForming) {
                                            // 动画未结束，绘制
                                            transform.draw(canvas, paint, isShaded);
                                        } else {
                                            // 动画结束，释放操作
                                            transform.recycle(heartPool);
                                            // 释放单个已完成动画
                                            animList.remove(transform);
                                        }
                                    }
                                }
                            }
                        }
                    } catch (Exception e) {
                        unlockCanvas = null;
                        e.printStackTrace();
                    } finally {
                        try {
                            if (canvas != null) {
                                surfaceHolder.unlockCanvasAndPost(canvas); // 结束锁定画图，并提交改变。
                            }
                        } catch (Exception e) {
                            try {
                                java.lang.reflect.Field field = SurfaceView.class.getDeclaredField("mSurfaceLock");
                                field.setAccessible(true);
                                ReentrantLock lock = (ReentrantLock) field.get(HeartAnimSurfaceView.this);
                                lock.unlock();
                            } catch (Exception ex) {
                                unlockCanvas = canvas;
                            }
                        }
                    }
                }
            }
            try {
                if (animList == null || animList.size() == 0) {
                    Thread.sleep(PAUSE_RENDER_DURATION); // 无内容时的刷新间隔
                } else {
                    // 绘制时间越久，绘制间隔越大，防止性能问题
                    sleep = RENDER_DURATION + (System.currentTimeMillis() - start) / 2;
                    if (sleep < RENDER_DURATION * 2) sleep = RENDER_DURATION * 2;
                    // 渲染间隔
                    Thread.sleep(RENDER_DURATION);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
这样飘星的动画就完成了。

### 参考
- [Android基础-秒懂SurfaceView的双缓冲机制](https://www.jianshu.com/p/556ca0c1889b)
- [SurfaceView的特性总结](https://www.jianshu.com/p/de6bd55b2df8)
- [SurfaceView的双缓冲机制](https://blog.csdn.net/rabbit_in_android/article/details/50807765)
- [SurfaceView和View对比](http://zhupeng.space/2016/10/26/surfaceview/)
- [SurfaceView的多线程绘图](https://www.jianshu.com/p/e81aacb365bc)

### 最后

SurfaceView动画由于绘制和生命周期的复杂性，还有对内存和Cpu的巨大消耗，导致其平时使用的场景不会很多，大致了解下原理即可，[所有的示例Demo都在这里哦，仅内部人员可以体验！](https://git.corp.plu.cn/AppComponentLibs/LongzhuAnim.git)

---

<img src="https://raw.githubusercontent.com/Chensigou/resource/master/logo.png" height="120" />
