---
title: Android性能优化-MultiDex优化
date: 2019-12-04 08:19:06
tags:
---

## 为什么需要MultiDex

在Android5.0以下，虚拟机执行的是Dalvik指令，Dalvik指令集的方法索引参数至占用两个字节，也就是最多能区分65536个不同的方法，这个是65536方法数限制的根本原因，当然dx工具在打包时会直接判断方法数是否超过65536，超过就直接抛异常，这个只是表面原因。ART本身就对MultiDex做了支持，因此无需开发者对其做特殊处理。


可以参考这篇文章：[Does the Android ART runtime have the same method limit limitations as Dalvik?](https://stackoverflow.com/questions/21490382/does-the-android-art-runtime-have-the-same-method-limit-limitations-as-dalvik/21492160#21492160 "https://stackoverflow.com/questions/21490382/does-the-android-art-runtime-have-the-same-method-limit-limitations-as-dalvik/21492160#21492160")


## 怎么使用MultiDex

Google官方提供了MultiDex支持库，使用起来很简单，首先引入依赖：

	dependencies {
      implementation 'com.android.support:multidex:2.0.0'
    }

然后把MultiDex开关打开：

	android {
        defaultConfig {
            ...
            multiDexEnabled true
        }
        ...
    }

最后在Application的attachBaseContext方法里初始化：

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);

        MultiDex.install(this);

    }

为什么要在attachBaseContext方法中初始化呢，在onCreate方法中行不行？这涉及到MultiDex的内部原理，简单概况下，APK运行的时候，Dalvik虚拟机只会帮我们加载classes.dex，而classes2.dex，classes3.dex等不会加载，MultiDex的install方法所做的工作就是加载这些剩余的dex，为了避免发生NoClassDefFoundError，我们应该尽早把dex加载进来，很明显attachBaseContext方法执行时机比onCreate方法早。

在实际开发中发现，install方法可能会比较耗时，这样就会导致App的启动速度变慢，甚至发生ANR，所以这里就有两个问题：

1. install方法具体做了什么工作，为什么会耗时？

2. 在保证dex能够加载进来的情况下，要如何提升App启动速度，避免发生ANR？

## MultiDex内部原理

直接看install方法源码：

    public static void install(Context context) {
        Log.i("MultiDex", "Installing application");
        if (IS_VM_MULTIDEX_CAPABLE) {
            Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
        } else if (VERSION.SDK_INT < 4) {
            throw new RuntimeException("MultiDex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
        } else {
            try {
                ApplicationInfo applicationInfo = getApplicationInfo(context);
                if (applicationInfo == null) {
                    Log.i("MultiDex", "No ApplicationInfo available, i.e. running on a test Context: MultiDex support library is disabled.");
                    return;
                }

                doInstallation(context, new File(applicationInfo.sourceDir), new File(applicationInfo.dataDir), "secondary-dexes", "", true);
            } catch (Exception var2) {
                Log.e("MultiDex", "MultiDex installation failure", var2);
                throw new RuntimeException("MultiDex installation failed (" + var2.getMessage() + ").");
            }

            Log.i("MultiDex", "install done");
        }
    }

如果是Android5.0+版本，就会走IS_VM_MULTIDEX_CAPABLE的分支，什么也不做，而Android5.0以下，最终会调用doInstallation方法：

    private static void doInstallation(Context mainContext, File sourceApk, File dataDir, String secondaryFolderName, String prefsKeyPrefix, boolean reinstallOnPatchRecoverableException) throws IOException, IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, SecurityException, ClassNotFoundException, InstantiationException {
        synchronized(installedApk) {
            if (!installedApk.contains(sourceApk)) {
                installedApk.add(sourceApk);
                if (VERSION.SDK_INT > 20) {
                    Log.w("MultiDex", "MultiDex is not guaranteed to work in SDK version " + VERSION.SDK_INT + ": SDK version higher than " + 20 + " should be backed by " + "runtime with built-in multidex capabilty but it's not the " + "case here: java.vm.version=\"" + System.getProperty("java.vm.version") + "\"");
                }

                ClassLoader loader;
                try {
                    loader = mainContext.getClassLoader();
                } catch (RuntimeException var25) {
                    Log.w("MultiDex", "Failure while trying to obtain Context class loader. Must be running in test mode. Skip patching.", var25);
                    return;
                }

                if (loader == null) {
                    Log.e("MultiDex", "Context class loader is null. Must be running in test mode. Skip patching.");
                } else {
                    try {
                        clearOldDexDir(mainContext);
                    } catch (Throwable var24) {
                        Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var24);
                    }

                    File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
                    MultiDexExtractor extractor = new MultiDexExtractor(sourceApk, dexDir);
                    IOException closeException = null;

                    try {
                        List files = extractor.load(mainContext, prefsKeyPrefix, false);

                        try {
                            installSecondaryDexes(loader, dexDir, files);
                        } catch (IOException var26) {
                            if (!reinstallOnPatchRecoverableException) {
                                throw var26;
                            }

                            Log.w("MultiDex", "Failed to install extracted secondary dex files, retrying with forced extraction", var26);
                            files = extractor.load(mainContext, prefsKeyPrefix, true);
                            installSecondaryDexes(loader, dexDir, files);
                        }
                    } finally {
                        try {
                            extractor.close();
                        } catch (IOException var23) {
                            closeException = var23;
                        }

                    }

                    if (closeException != null) {
                        throw closeException;
                    }
                }
            }
        }
    }

代码比较多，但思路很清晰，分为两步：

1. 先是读取dex文件
	
	这里会去解压apk，遍历读取里面的classes2.dex, classes3.dex等等，然后再压缩成classes2.zip, classes3.zip等等。注意这里是有缓存的，如果之前已经处理过了就直接读取文件了，当然读取文件也是比较耗时的。

2. 安装到classloader中
	
	这里其实就是把dex追加到classloader的dexElements数组中，以后查找类，就可以找到了。

很明显，第一个操作涉及到文件的解压，读取，压缩，会比较耗时，在UI线程中做这么耗时的操作，肯定是有问题的；而第二个操作设计到反射等调用，也是比较耗时的。

## 优化MultiDex

从上面看的，MultiDex的install方法比较耗时，特殊是在apk首次运行的时候，需要对其进行优化。

### 方法一：子线程并行加载

直接把install方法放到子线程中执行，但这时UI线程会继续执行下去，如果遇到还没加载进来的类就会导致NoClassDefFoundError，当然我们可以手动分dex，把先用到的类都打到主dex里，包括所有的ConentProvider等，这方法听起来就不太靠谱，需要自己去维护一个列表，各种代码路径都会覆盖到，容易出错，风险会比较大。

### 方法二：临时进程阻塞加载

既然子线程加载不靠谱，就还是得回到原来的阻塞UI加载，但我们改为开一个临时进程进行加载，这样就不会导致主进程发生ANR，而且临时进程中采用异步加载，也不会发生ANR。具体的方案如下：

1. 主进程创建临时文件，用来给主进程判断临时进程是否已经加载完成。

2. 启动运行在临时进程的Activity(Loading)，在该Activity里异步调用MultiDex的install方法加载dex，加载结束后，删除临时文件，关闭当前Activity。

3. 主进程开启死循环，检测临时文件是否不存在，如果不存在，就跳出循环。注意这里本质上还是会阻塞UI线程，但因此当前进程已经不再是前台进程，不会发生ANR，这是关键点。

4. dex加载结束后，主进程继续执行下面的操作。


## 参考资料

Android拆分与加载Dex的多种方案对比：[https://cloud.tencent.com/developer/article/1030669](https://cloud.tencent.com/developer/article/1030669 "Android拆分与加载Dex的多种方案对比")

MultiDex（一）之源码解析：[https://cloud.tencent.com/developer/article/1190958](https://cloud.tencent.com/developer/article/1190958 "MultiDex（一）之源码解析")

Multidex（二）之Dex预加载优化：[https://cloud.tencent.com/developer/article/1190961](https://cloud.tencent.com/developer/article/1190961 "Multidex（二）之Dex预加载优化")

MultiDex（三）之异步加载优化：[https://cloud.tencent.com/developer/article/1190962](https://cloud.tencent.com/developer/article/1190962 "MultiDex（三）之异步加载优化")