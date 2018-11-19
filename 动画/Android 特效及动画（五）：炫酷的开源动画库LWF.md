
## 第五部分 - 第三方动画库

> Android 特效及动画，从入门到放弃，希望不是这个结果，哈哈。

### 概述

近几年市面上涌现出了很多热门的动画库，比较优秀的有 **Lottie** 和 **SVGA** 了，下面来简单对比一下：

| 动画库 | Lottie | Svga | Lwf |
|---|---|---|---|
| 支持平台 | Android/iOS/Web | Android/iOS/Web | Android/iOS/Web |
| 设计工具支持 | After Effects | AE & Flash | Flash |
| 设计成本 | 需要命名规范 | 无 | 需要转换 |
| 资源包大小zip | 中 | 很小 | 大 |
| 动画复杂性&流畅度 | 高 | 高 | 极高 |
| 性能消耗 | 中 | 中 | 高 |
| 官网 | [地址](http://svga.io/) | [地址](https://airbnb.design/lottie/) | [地址](https://github.com/Chensigou/pure2d)

总结一下lottie和svga虽然底层的实现方式不同，但是最终还是基于View的绘制，性能表现着重处理了图片资源的复用，同时简化动画的图层和高阶特效，所以在内存、Cpu、Gpu上的表现都还不错，远好于帧动画。而 **Lwf** 这个库底层是基于 **GlSurfaceView** 来实现的，基于SurfaceView的特点就是无论多复杂的动画，流畅性都可以得到保障，缺点就是在Cpu和内存上的占用也很大。

### Lwf动画
基于多方面的考虑（先入为主，更换难度是一个较重要的考量，当前已上线的道具资源重做的成本，另一个是性能方面的考量，直播房间中的动效已经应用的相当多了也很复杂，基于View的动画在人气爆满的房间里还是会有流畅性的问题），我们最终还是选择了Lwf这个开源动画框架来实现大额付费道具的动画；

<img src="https://github.com/Chensigou/resource/blob/master/video2gif_20180831_194952.gif?raw=true" height="320" />

#### 1. 简介

LWF is an animation engine which can play animation data converted from FLASH contents in HTML5, Unity, Cocos2d-x, iOS UIKit, and more. LWF is designed to make game development easy and fun.

> 简单的讲，LWF把由的AdobeFlash生成的SWF动画数据，转换为唯一的格式的框架，使其可以在Unity和HTML5播放。LWF可以减少对游戏中使用的2D角色动画或2D用户界面的开发成本，是一种小成本但很高效的动画框架。

#### 2. 动画库

动画库的核心类了解：

- BaseStage extends GLSurfaceView // 整个动画的容器，继承GLSurfaceView

- Scene extends Renderer // 动画渲染器，继承Renderer，实现了lwf渲染，mScene.addChild(mLWFObject)；

- LWFManager // lwf动画数据控制器，控制创建和释放所有的LWFData和LWF对象

- LWFObject // lwf动画内容实例，整合LWFData数据生成动画控制的LWF类，mLWF = mLWFObject.attachLWF(mLWFData)；

- LWFData // lwf动画内容元素集合，包含xxx.ani、xxx1.png、xxx2.png...

- LWF // lwf动画控制类，包含位移、缩放、开始暂停等控制操作，mLWF.scale(），mLWF.moveTo()，mLWF.addEventHandler()；

- AssetTexture & FileTexture // 指定路径下的文件，将xxx.png生成GL.Texture纹理，mLWFData.setTextures(textures)；

相互关系图如下：

```
graph TB
BaseStage-->Scene
Scene-->LWFObject

LWFObject-->LWFData
LWFObject-->LWF

LWFManager-->LWFData
LWFManager-->LWF

LWFData-->AssetTexture
LWFData-->FileTexture
```

实现原理总结，将动画元素转换成了GL纹理，将动画轨迹文件和纹理封装成动画对象，重写了GL的Render渲染，将动画对象丢进去，底层根据既定的规则，还原成一个完整的动画。

#### 3. 动画文件转换及制作

转换和制作步骤较复杂，这里不做详细说明，[可以参考这篇](https://github.com/Chensigou/resource/blob/master/Flash%20%E5%88%B6%E4%BD%9C.md)。

#### 4. 组件业务接口设计

组件接口设计，主要是对动画位置的配置进行了一定封装，增加了一些基础的控制操作，最新增加了动态添加/删除布局操作。

```
LwfConfigs //动画配置类
HorizontalSite/VerticalSite //位置配置类
Arg //动画常量参数，包含动画标签、类型等，回调时返回用于做匹配

void showLwfStatus(boolean b); //是否打印log

void setDefaultOrientation(boolean isPort); //设置初始方向

void setSurfaceCallback(LwfStateEvent callback);

boolean isQueueEmpty(); //队列中的动画是否全部完成

boolean isPaused(); //当前surface是否暂停

boolean isViewAdded(); //当前view是否添加

void resume(); //恢复播放

void pause(); //暂停播放

void setLwfVisible(boolean v); //设置动画是否可见

void clearLwfOnGlThread(final LWFObject lwfObject); //清除指定/所有 LWF动画

void release();

void addViewTimely(LwfGLSurface.LwfViewAddListener listener); // 立即添加view

void removeViewTimely(); // 立即移除view

void startLwfAnimation(final LwfConfigs lwfConfigs, final boolean canOverlay);
```

动画的配置参数可以通过下发的方式来设置，增加灵活性，最终设置配置的伪代码：

```
//设置动画配置
LwfConfigs lwfConfigs = new LwfConfigs();
//动画核心文件路径
lwfConfigs.setLwfPath(path);
//纹理图片路径
lwfConfigs.setTexPath(textPath);
//sdk还是asset存储源
lwfConfigs.setFromAsset(true);
//动画大小{屏幕高度的百分比， 屏幕宽度的百分比}
lwfConfigs.setScale(new float[]{1f, 0});
//动画位置Site：gravity（left，center，right），margin（对应方向屏幕百分比）
lwfConfigs.setPortHorizontalSite(Site).setPortVerticalSite(Site)
            .setLandHorizontalSite(Site).setLandVerticalSite(Site);
```

基本功能实现的伪代码：

```
//可选，自定义文案
lwfGLSurface.setSelfTextPath("").setSelfTextName("")
//可选，设置当前动画标签
lwfGLSurface.setArg(Arg)
//开始播放
lwfGLSurface.startLwfAnimation(lwfConfigs);
//设置事件监听
lwfGLSurface.setSurfaceCallback(new LwfStateEvent() {
    void onAnimStatus(boolean isStatus, String s1, String s2, String s3) //动画状态，动画总数，帧率，纹理总数
    void onAnimComplete(Arg arg) //动画播放完成
    void onAnimError(Arg argue, int errorType) //动画加载出错
    void onAnimClick(Arg argue) //动画被点击事件
    void onAnimHold(Arg argue, final HoldCallback callback) //动画hold态，处理接口请求等
});
```

自定义纹理的方案：

简单的将就是先预埋帧透明纹理，在加载时动态hook替换掉真正需要的纹理（在动画增加一帧透明块儿，专门放置文字，动画导出后这个透明块儿就是那张透明的长方形图片，在展示动画前，通过代码在图片上绘制需要的文字，保存到本地，在加载动画的texture时，读取到这张长方形图片时，就去用重新绘制的那张图片替换长方形图片）

<img src="https://github.com/Chensigou/resource/blob/master/WechatIMG1.png?raw=true" height="48" />



    
### 参考
- [Lwf 官方介绍](http://gree.github.io/lwf/)
- [Lwf 官方Demo](https://github.com/splhack/Hello-LWF-Cocos2d-x)

### 最后

关于动画库的选择还是要考虑全面些，像 **Lottie** 和 **SVGA** 的框架还是很不错的，和 **Lwf** 相比会轻很多，并且View的层级和位置控制也会很方便，[三个框架的示例Demo都在这里哦，仅内部人员可以体验！](https://git.corp.plu.cn/AppComponentLibs/LongzhuAnim.git)

---

<img src="https://raw.githubusercontent.com/Chensigou/resource/master/logo.png" height="120" />
