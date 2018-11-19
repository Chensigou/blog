
## Gradle 命令行的基本使用

- [命令行的基本使用、图形用户界面、编写构建脚本](http://wiki.jikexueyuan.com/project/gradle/using-the-gradle-command-line.html)
- [开发命令行完全攻略](http://www.ouchangjian.com/article/583c4775c02117735ff7bad5)
- [Error: Connection timed out: connect](http://blog.csdn.net/myatlantis/article/details/72779929)
- [Gradle常用指令](https://github.com/shekhargulati/gradle-tips)

- ### Gradle 基本指令

```
gradlew tasks               :查看可用任务列表
gradlew tasks --offline     :以最快速度执行单元测试
gradlew tasks --all         :查看所有任务
gradlew clean               :清理專案
gradlew build               :建置tasks

// 查看xxx Module的依赖树详情
gradlew dependencies :xxx:dependencies
gradlew dependencies :xxx:dependencies >log.txt :依赖树详情保存至当前目录的log文本下
// 检测module的comiple情况
gradlew -q dependencies <modelName>:dependencies --configuration compile
// 解决gradlew: Permission Denied的问题
chmod +x gradlew

// 注：MAC需要在 gradlew前面加 (./)

```

## Gradle版本问题

1. [Gradle最全概念整合](http://hucaihua.cn/2016/09/27/Gradle_upgrade/)---Gradle概念介绍，gradle版本对应关系，以及gradle的各种问题解决

2. [Gradle Distributions](http://services.gradle.org/distributions/)---Gradle版本下载

3. [Android Plugin for Gradle Release Notes](https://developer.android.google.cn/studio/releases/gradle-plugin.html)---Gradle版本对应

## 方法数查询

1. [完整App方法数查询](https://blog.csdn.net/sodino/article/details/56482070)

2. [针对Jar方法数查询](https://www.jianshu.com/p/cf97b6e78214) --- [[参考]](https://blog.csdn.net/lulicheng/article/details/78665505)**（使用'./dx'来执行shell文件）**


## Maven/JCenter仓库介绍

- [Android Studio 发布Java项目到Maven/JCenter仓库](https://liungkejin.github.io/2016/03/27/Publish-AAR-jcenter.html)


## Gradle Android Plugin 中文手册

- [Gradle Android Plugin 中文手册](https://chaosleong.gitbooks.io/gradle-for-android/content/)