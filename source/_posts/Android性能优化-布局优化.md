---
title: Android性能优化-布局优化
date: 2019-11-30 22:49:38
tags:
---

## 为什么需要优化

Android布局可能存在的主要问题有两个：过度绘制和卡顿，过度绘制会浪费资源，卡顿会影响用户体验，甚至导致ANR，因此需要进行优化，优化都是围绕着这两个问题展开的，最终目的就是减少过度绘制，减少卡顿。

## 常见检测工具

有时通过直觉无法明显发现过度绘制和卡顿，或者无法准确量化其严重程度，因此需要借助检测工具来帮助我们排查问题，常用的工具主要有两个：开发者选项中的调试GPU过度绘制和GPU呈现模式分析。

* **开发者选项-调试GPU过度绘制**

	这里有五种颜色，代表过度绘制的严重程度

	真彩色：没有过度绘制
	
	蓝色：过度绘制1次

	绿色：过度绘制2次

	粉色：过度绘制3次

	红色：过度绘制4次或更多
	
	优化的目标，应该是让尽量多的区域为真彩色和蓝色。

	官方文档：[https://developer.android.google.cn/topic/performance/rendering/overdraw](https://developer.android.google.cn/topic/performance/rendering/overdraw "https://developer.android.google.cn/topic/performance/rendering/overdraw")

* **开发者选项-GPU呈现模式分析**

	开启后屏幕底下会有不断移动的柱状图，一条代表一帧，高度代表该帧处理和渲染绘制整个过程花费的总时间，低于水平绿线（16ms）才不会出现卡顿。
	对于每一帧，不同颜色代表不同阶段所花费的时间，这里有八种颜色，包括：

	绘制时间：onDraw执行时间
	
	测量和布局时间：onMeasure和onLayout执行时间

	动画时间：ObjectAnimator等动画执行时间，常见场景比如ListView等列表的fling动画。

	CPU等GPU缓冲区时间：时间长，说明GPU工作太多

	输入处理时间：处理用户输入的时间，如果时间太长需要开子线程处理。

	构建DisplayList时间：构建绘制命令时间

	位图信息上传给GPU时间：将位图信息上传到GPU所花的时间

	VSync延迟时间：VSync处理延迟时间，时间太长说明UI线程执行了耗时操作

	官方文档：[https://developer.android.google.cn/topic/performance/rendering/profile-gpu.html](https://developer.android.google.cn/topic/performance/rendering/profile-gpu.html "https://developer.android.google.cn/topic/performance/rendering/profile-gpu.html")

## 常见原因和优化方法

* **在UI线程执行耗时操作**

	**原因：** 比如读写SharePreferences，读写数据库等。

	**解决方法：** 耗时操作都应该放到子线程中执行。


* **布局过于复杂**

	**原因：** 布局嵌套层数过深，以及布局过于复杂，measure，layout，draw需要花费大量时间。

	**解决方法：** 使用merge标签合并布局，减少一层嵌套；使用ViewStub实现懒加载布局，必要时才inflate布局；使用RelativeLayout减少嵌套层数，相同层数优先用LinearLayout而不是RelativeLayout。


* **过度绘制**

	**原因：** 同个位置设置了重复背景，同个像素被绘制多次。

	**解决方法：** 去除重复背景，去除Window默认背景，自定义View通过Canvas的clipRect指定绘制区域。


* **频繁GC导致UI线程被挂起**

	**原因：** 内存抖动。

	**解决方法：** 排查大量创建短命对象的地方。


* **频繁刷新整个UI布局**
 
	**原因：** 整个UI布局的measure，layout，draw耗时较多。

	**解决方法：** 只刷新需要刷新的区域即可，比如调用notifyItemChanged刷新RecyclerView某个Item，ListView也类似，最好能拿到某个Item的View进行刷新。



