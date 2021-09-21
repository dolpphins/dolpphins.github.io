---
title: Android性能优化-APK瘦身
date: 2019-12-05 08:18:06
tags:
---

## 前言

APK包过大，可能会导致用户不想下载，或者消耗用户更多的流量，需要更多的下载等待时间，因此需要对APK进行瘦身，使得APK包尽量精简。

## 瘦身方法

* **so优化**
	
	1. so裁剪：避免为了使用一个小功能引入一个大的so，删除用不到的so代码和依赖。
	
    2. 只保留armeabi或者armeabi-v7a下的so文件：armeabi属于比较老的CPU架构，不支持硬件浮点运算，所以速度较慢，但兼容设备更多，而现在大部分首页都是支持armeabi-v7a架构的。
    
	3. 分渠道打包so：不同渠道的用户手机CPU架构不同，为了避免同时加入多种so，可以对不同渠道放入对应的so。

* **依赖库**

	这里主要指第三方的jar，aar等依赖库，引入的库应该尽可能精简。

* **res优化**

	1. 只保留一套图片： 对于用到的图片，一般放到xxhdpi目录就可以，具体放哪里，要看目标用户手机主要的分辨率是多少。

	2. 动态加载：不是很重要的图，或者非首屏显示的图，改为动态加载。

	3. 图片压缩：tinypng，用webp代替png。

	4. 无用资源：用AS删除无用资源。

	5. 矢量图：用矢量图代替真实图片，矢量图代码占用空间小，而且不会失真。
	
	6. 多语言：对国内用户只打中文包，resConfigs指定为zh-rCN。

	7. 使用AndResGuard开源库：微信出的工具，用于压缩资源路径，同时也起到混淆资源文件名的作用。

	8. 精简字体文件：字体文件只包含需要用到的文字、数字或者符合等，而不是直接引入整个字体文件。

* **assets优化**

	跟res优化差不多，删除无用资源，压缩资源，动态下载。

* **代码优化**

	1. 打开minifyEnabled：代码压缩，编译时会自动移除没有被使用到的代码，可以通过keep来强制指定不移除某些代码。

	2. 打开shrinkResources：资源压缩，必须在minifyEnabled为true的前提下使用，编译时会自动移除没有被使用到的资源，也可以自定义指定哪些资源不移除。

	3. proguard代码混淆：这个正式包都有，代码混淆能够减少类名、方法名等长度，移除没有被使用到的代码。

		注意minifyEnabled和proguard的区别：minifyEnabled只能移除无用代码，速度较快；而proguard除了能移除无用代码，还可以进行名称混淆，优化代码，速度较慢。参考文章：[https://stackoverflow.com/questions/37007485/whats-the-difference-between-minifyenabled-and-useproguard-in-the-android-p](https://stackoverflow.com/questions/37007485/whats-the-difference-between-minifyenabled-and-useproguard-in-the-android-p "What's the difference between “minifyEnabled” and “useProguard” in the Android Plugin for Gradle?")

	4. 代码动态化：组件化，代码动态下载。

	5. 单个枚举大概占用1~1.4KB，用常量代替枚举。