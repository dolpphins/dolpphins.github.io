---
title: Android性能优化-内存优化
date: 2019-12-01 09:29:31
tags:
---

## 为什么需要优化

Android中内存优化是一个比较重要的话题，因为内存会直接影响到程序的运行性能，在开发中经常会出现一些内存相关的问题，包括内存泄露，内存抖动，占用过多内存等，因此优化的目的，就是让减少这几类问题的出现，或者彻底解决，让程序运行起来更加流畅，性能更高效。

## 常用检测工具

* **LeakCanary**

	用于检测内存泄露，如果发生内存泄露，会给出泄露对象的引用链，其内部是通过WeakReferences的回收队列实现内存泄露检测的。

* **Android Studio的Profiler工具**

	可查看对象占用内存情况，只能查看debuggable应用。

* **adb命令**

	执行命令adb shell dumpsys meminfo，可以查看所有应用的内存信息。常用的参数有--package，用于指定要查看的包名的所有进程的内存信息，比如我们要查看微信的内存信息，执行下面的命令：

		adb shell dumpsys meminfo --package com.tencent.mm

	这里就会微信所有进程的内存信息，如果我们只需要看主进程的信息，可以直接通过pid指定，所以先执行命令ps获取到pid：

		adb shell ps -A | grep com.tencent.mm
	
	打印结果如下：

		u0_a170       3525   633 2068856  21644 0                   0 S com.tencent.mm:appbrand1
		u0_a170      19179   633 2577364 112404 0                   0 S com.tencent.mm
		u0_a170      19404   633 2240664  11740 0                   0 S com.tencent.mm:appbrand0
		u0_a170      26971   633 1965968   5312 0                   0 S com.tencent.mm:exdevice
		u0_a170      28292   633 2003436   5296 0                   0 S com.tencent.mm:push

	看到主进程的pid为19179，然后执行命令：

		adb shell dumpsys meminfo 19179

	打印结果如下：
		
		Applications Memory Usage (in Kilobytes):
		Uptime: 138531332 Realtime: 241333585
		
		** MEMINFO in pid 19179 [com.tencent.mm] **
		                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
		                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
		                ------   ------   ------   ------   ------   ------   ------
		  Native Heap    40613    40564        0    72101   164480   132995    31484
		  Dalvik Heap    19065    19016        0     8873    52069    27493    24576
		 Dalvik Other    11200    11196        0      893                           
		        Stack      100      100        0       16                           
		       Ashmem        4        4        0        0                           
		      Gfx dev     6432     6432        0        0                           
		    Other dev       28        0       28        0                           
		     .so mmap     6138      440     4328     3370                           
		    .jar mmap       28       28        0      336                           
		    .apk mmap     1157       88      672      160                           
		    .ttf mmap       60        0        0        0                           
		    .dex mmap    22720        0    19372       28                           
		    .oat mmap     2520        0        4        0                           
		    .art mmap    10700     9644      260     3067                           
		   Other mmap      336        0      324        4                           
		   EGL mtrack     9248     9248        0        0                           
		    GL mtrack     5436     5436        0        0                           
		      Unknown     2666     2664        0     4830                           
		        TOTAL   232129   104860    24988    93678   216549   160488    56060
		 
		 App Summary
		                       Pss(KB)
		                        ------
		           Java Heap:    28920
		         Native Heap:    40564
		                Code:    24932
		               Stack:      100
		            Graphics:    21116
		       Private Other:    14216
		              System:   102281
		 
		               TOTAL:   232129       TOTAL SWAP PSS:    93678
		 
		 Objects
		               Views:     1388         ViewRootImpl:        1
		         AppContexts:        7           Activities:        1
		              Assets:        7        AssetManagers:        0
		       Local Binders:      157        Proxy Binders:       98
		       Parcel memory:       82         Parcel count:      327
		    Death Recipients:        8      OpenSSL Sockets:        0
		            WebViews:        0
		 
		 SQL
		         MEMORY_USED:      528
		  PAGECACHE_OVERFLOW:       64          MALLOC_SIZE:      117
		 
		 DATABASES
		      pgsz     dbsz   Lookaside(b)          cache  Dbname
		         4      436             97       25/40/10  /data/user/0/com.tencent.mm/databases/beacon_tbs_db
		         4      108            109       27/42/16  /data/user/0/com.tencent.mm/databases/google_app_measurement.db
		         4       36             99         4/34/6  /data/user/0/com.tencent.mm/databases/tes_db
		 
		 Asset Allocations
		    : 132K

	对于第一部分，一般只需要关注前两列就行，Pss Total表示该进程占用内存大小，包括了按比例计算进程间共享内存，而Private Dirty就只是该进程占用内存大小。然后对应到具体的行，Native Heap为本地堆占用内存大小，Dalvik Heap为Java堆占用内存大小，Dalvik Other为JIT和垃圾回收记录占用内存大小，.so mmap为映射的so代码占用的内存大小，.dex mmap为映射的dex代码占用的内存大小。

	对于第三部分，表示在当前进程中，某些重要类型的实例对象数量，比如ViewRootImpl就是窗口数量，AppContexts就是Context数量，而Activities为Activity的数量。


## 内存泄露问题

* **Cursor没有关闭**

	**原因：** 会导致Cursor使用的共享内存块泄露。

	**解决方法：** 在finally语句块中关闭Cursor。


* **监听器没有反注册**

	**原因：** 监听器对象会被一直引用着无法被回收。

	**解决方法：** 在对应的生命周期方法中进行反注册。


* **static变量持有对象**

	**原因：** static变量的生命周期很长，会导致被引用的对象无法被回收。这里的static变量包括单例，普通static对象等。

	**解决方法：** 避免static变量引用短生命周期的对象，及时清空static变量引用，采用Application对象代替Activity对象作为上下文。


* **非静态内部类**

	**原因：** 非静态内部类对象会默认持有外部类对象的引用，如果该内部类对象被长期引用，就会导致外部类对象无法被回收。

	**解决方法：** 将内部类声明为静态的，如果需要调用外部类方法等，则采用WeakReferences实现。


* **WebView**
 
	**原因：** 页面关闭但内存资源不会被释放等。

	**解决方法：** 在临时进程中使用WebView，可能需要处理好进程间通信问题。


* **5.0以前AlertDialog**
	
	**原因：** 触发点击后事件会被包装成Message对象发送出去，Message对象持有事件监听器对象引用，如果事件监听器对象持有外部类对象引用，而且Message对象阻塞在消息队列中，就会导致该引用链上的对象无法被回收。

	**解决方法：** 采用静态Handler+WeakReferences优化。 

* **循环动画**

	**原因：** 退出页面后动画还在播放，导致Activity无法被回收。

	**解决方法：** 退出页面时停止动画。

## 内存抖动问题

1. 避免在循环中创建对象（包括执行字符串+号拼接）。

2. 避免在onDraw中创建对象。

3. 通过复用减少对象的创建和回收次数，如复用Bitmap等。


## 减少内存占用
	
1. 对于数量级小于1000，使用SparseArray和ArrayMap代替HashMap。

2. 使用StringBuilder进行字符串拼接。

3. 尽量使用基本类型，而不是使用其对应的包装类。

