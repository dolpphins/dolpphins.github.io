---
title: Android性能优化-启动速度优化
date: 2019-12-03 08:44:10
tags:
---

如果APP启动速度很慢，会导致用户体验很差，甚至用户流失。所以我们需要持续地对启动速度进行优化，不断减少启动等待时间，把优化做到极致。

## 启动类型

启动类型可以分为三种：

* 冷启动

	进程不存在，启动时需要先创建进程，然后再创建并打开页面。

* 温启动

	进程存在，但页面不存在，只需要创建并打开页面。

* 热启动

	进程存在，页面也存在，直接把页面拉到前台即可。

## 检测工具

* 代码日志
	
	通过代码计算时间差，得到代码执行时间，存在的问题是需要手动添加计算代码，如果要检测的地方很多，就需要加很多计算代码，很不方便。

* 系统日志

		ActivityManager: Displayed com.tencent.mm/.app.WeChatSplashActivity: +424ms (total +1s37ms)
	
	上面是微信冷启动的系统日志，这里涉及到启动页和首页两个Activity的启动，总时长为1s27ms，而启动首页占用的时长为424ms，时长是在从框架层开始启动Activity（包括创建进程）到Activity被绘制完成这一过程所需要的时间。

	
		ActivityManager: Displayed com.tencent.mm/.ui.LauncherUI: +288ms
	

	这个是微信温启动（进程还在，Activity不在）的系统日志，可以看到明显比冷启动快了。

	注意，这个系统日志，在Android4.4开始才有。


* adb shell am start -W 包名/Activity全限定名	

		Starting: Intent{act=android.intent.action.MAIN category[android.intent.category.LAUNCHER] cmp=com.tencent.mm/.ui.LauncherUI}
		Status: ok
        Activity: com.tencent.mm/.ui.LauncherUI
		ThisTime: 1138
        TotalTime: 1138
        WaitTime: 1168
        Complete

	这里有三个数值，ThisTime表示启动目标Activity的时长，TotalTime表示最终启动目标Activity的时长，WaitTime则在TotalTime上加上了等待时长，也就是startActivityAndWait方法的执行时间。

* TraceView

	这个工具用起来比较简单，代码如下：
    
		Debug.startMethodTracing("weixin");
		......
		Debug.stopMethodTracing();
	
	然后就可以去SD卡根目录（记得开SD卡权限）找到weixin.trace文件，导入AS分析。
	
	用这个工具，主要是用于分析APP启动时哪些方法执行时间比较长，定位启动慢的代码位置。

	TraceView有一个问题，就是会导致App很卡，开销太大，因为它会收集所有方法的执行时间。

	另外，AS的Profiler工具，也可以直接生成trace文件。当然，只能用于可debug的App进程。

* systrace

	参考文章：[https://zhuanlan.zhihu.com/p/27331842](https://zhuanlan.zhihu.com/p/27331842 "https://zhuanlan.zhihu.com/p/27331842")

	google在Framework层的关键过程加了监控，比如启动Activity流程中Framework层的调用过程，可以直接获取到其运行时长数据，这个TraceView做不了。

	另外，Android P在开发者选项中新增了系统追踪，根据官方介绍，功能跟systrace差不多，不用连接USB即可获取到trace文件，具体使用方法和原理等有空研究。

## 优化方法

先看一下冷启动Activity的流程：

1. startActivity(Launcher)
2. startActivity(AMS)
3. fork进程
4. attach(ActivityThread)
5. handleBindApplication(ActivityThread)
6. **attachBaseContext(Application)**
7. **installContentProviders(ActivityThread)**
8. **onCreate(Application)**
9. UI线程开始loop循环
10. **onCreate(Activity)**

这里有些过程我们无法干预，事实上这个无法干预的过程也不是启动慢的主要原因，能干预的，就是attachBaseContext，installContentProviders，Application的onCreate，Activity的onCreate。

* **attachBaseContext**

	针对5.0以下的手机，一般我们很喜欢在这个方法里面做Multidex的初始化，理由是dex越早加载越好，而这个attachBaseContext方法，就是在pplication对象刚被创建后回调的。但Multidex的初始化，有可能是比较耗时的，关于如何解决MultiDex初始化耗时的问题，后面有专门的文章分析。

* **installContentProviders**

	应用进程启动后就会安装所有的ContentProvider，回调其onCreate方法，这样就会拖慢App启动速度，其官方文档说明如下：

>     
    /** 
	 * Implement this to initialize your content provider on startup.
     * This method is called for all registered content providers on the
     * application main thread at application launch time.  It must not perform
     * lengthy operations, or application startup will be delayed.
     *
     * <p>You should defer nontrivial initialization (such as opening,
     * upgrading, and scanning databases) until the content provider is used
     * (via {@link #query}, {@link #insert}, etc).  Deferred initialization
     * keeps application startup fast, avoids unnecessary work if the provider
     * turns out not to be needed, and stops database errors (such as a full
     * disk) from halting application launch.
     *
     * <p>If you use SQLite, {@link android.database.sqlite.SQLiteOpenHelper}
     * is a helpful utility class that makes it easy to manage databases,
     * and will automatically defer opening until first use.  If you do use
     * SQLiteOpenHelper, make sure to avoid calling
     * {@link android.database.sqlite.SQLiteOpenHelper#getReadableDatabase} or
     * {@link android.database.sqlite.SQLiteOpenHelper#getWritableDatabase}
     * from this method.  (Instead, override
     * {@link android.database.sqlite.SQLiteOpenHelper#onOpen} to initialize the
     * database when it is first opened.)
     *
     * @return true if the provider was successfully loaded, false otherwise
     */
    public abstract boolean onCreate();

* **Application的onCreate**

	 一般应用都会在这里做各种初始化，建议是对于非必要的初始化，可以改为用延迟初始化和子线程初始化。

* **Activity的onCreate**
	
	这里主要是布局过于复杂，导致inflate时间过长，可以根据UI优化建议进行优化。

## 保底方案

* 障眼法

	当做了上面的优化后，因为部分逻辑是非执行不可，App启动速度还是会比较慢，这时就需要采用一些类似障眼法的方案来优化用户体验。这里主要的方法是先跳转到启动页，启动页设置其主题背景为某张启动图片，广告信息，等初始化完成了，再跳转到主页面。
