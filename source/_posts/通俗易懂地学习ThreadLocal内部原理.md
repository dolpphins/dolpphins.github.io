---
title: 通俗易懂地学习ThreadLocal内部原理
date: 2020-01-24 11:27:51
tags:
---

### 前言

ThreadLocal是JDK中的一个类，很多基础框架和平时开发中都会使用到，因此有必要弄清其内部原理，才能更好地使用它。

### 使用方法

要弄清原理，还是要先知道如何使用，ThreadLocal用起来是很简单的，一般都是把ThreadLocal定义为static变量，也就是只有一个实例对象，如下：

	private static ThreadLocal<Integer> sThreadLocal = new ThreadLocal<Integer>();

然后就可以在各个线程中使用这个sThreadLocal了：

	for (int i = 0; i < 5; i++) {
		final int index = i;
		new Thread() {
			
			public void run() {
				sThreadLocal.set(index);
				
				try {
					Thread.sleep(1000);
				} catch (Throwable t) {
					t.printStackTrace();
				}
				
				System.out.println(sThreadLocal.get());
			}
			
		}.start();
	}

可以看到，一个ThreadLocal只能用来保存一个对象，如果需要保存多个对象，就需要定义多个ThreadLocal。

### 使用场景

ThreadLocal的作用就是用来保存线程相关的局部对象，当某个对象跟线程有一对一的关系，就可以使用ThreadLocal进行保存。

在Android中，我们知道一个线程对应一个Looper，因此Looper内部就使用到了ThreadLocal，把线程的Looper对象保存在ThreadLocal中。又比如AnimationHandler类，它内部也定义了一个ThreadLocal对象，用于保存线程的AnimationHandler对象，也就是说，线程的动画执行最终都是在被同一个AnimationHandler对象处理的，有兴趣可以自己看源码。

我们还注意到，定义的ThreadLocal虽然在多个线程中被使用，但不会有线程问题，因为每个线程访问的都是自己的局部对象。

### 原理分析

* 整体原理图

	![](/images/threadlocal.png)

	这个Thread就是线程对象，每个线程只有一个，可以通过Thread.currentThread()方法拿到，也可以在new Thread()时拿到，不管什么方式拿到的都是同个对象，这个对象里面有一个哈希表，每一个Entry的key就是ThreadLocal的弱引用，而value就是set进来的局部对象。

	可以看到这个哈希表是线程唯一的，而里面的每一个Entry，对应一个ThreadLocal对象，比如说我在线程A中用到了3个ThreadLocal，都设置了不同的局部对象，那么这个哈希表就有3个Entry对象。

* ThreadLocalMap

	上面所说的线程Thread对象中的哈希表，其真实类型就是ThreadLocalMap，它其实是一个简化版哈希表，初始容量为16，当然，只有真的放东西Entry数组才会初始化，threshold为容量的2/3，超过了就触发扩容，扩容也是比较简单的，就是直接扩大为原容量的2倍，至于哈希冲突问题，如果冲突就采用线性探测的方法解决。

	这里还有个问题，就是如何根据key确定放到哪个桶里，可以看下代码：

		int i = key.threadLocalHashCode & (len-1);

	threadLocalHashCode在初始化时被赋值：

		public class ThreadLocal<T> {
			......
			private final int threadLocalHashCode = nextHashCode();

			private static final int HASH_INCREMENT = 0x61c88647;

	    	private static int nextHashCode() {
	        	return nextHashCode.getAndAdd(HASH_INCREMENT);
	    	}
			......
		}

	也就是说，这个threadLocalHashCode，就是0x61c88647的整数倍，然后跟len-1进行与操作，就得到了桶的下标，至于为什么是0x61c88647这个数值，网上有分析文章，有兴趣可以自己查看。

* 内存泄漏

	根据上面的整体原理图，可以得到一条引用链：

		Thread对象 --> threadLocals --> Entry[] --> WeakReference<ThreadLocal<?>>和value

	因为Thread对象的生命周期是比较长的，只有当线程退出后，Thread对象才会被回收，那么在线程退出前，我们的ThreadLocal被WeakReference包着，如果外部没有强引用，在内存不足时gc会自动回收了，但那个value就不会了，需要我们自己手动去清空引用。

	因此这里存在一个内存泄漏的问题，如果某个局部对象使用ThreadLocal保存了，然后用完之后没有清除掉，线程又还没退出，就可能导致内存泄漏，当然解决的方法也很简单，调用remove方法就可以：

		sThreadLocal.remove();

	另外，并不是说这种场景下我们不调用remove方法，value引用就一直不会被置空，ThreadLocal内部做了优化，在get和set时会主动置空那些key被gc自动回收的value引用，不过一般我们的ThreadLocal都是定义为static，这种情况就不可能会被gc自动回收了。

* Netty的FastThreadLocal

	如果同个线程用到了多个ThreadLocal，也就是Entry[]会有多个元素，那么每次在搜索某个ThreadLocal对应在哪个桶这个过程就会比较耗时，特别是对于服务端并发量很大的情况下，性能损耗会很明显，因此FastThreadLocal就做了优化，直接把ThreadLocal的哈希表去掉了，改为用变量index记录当前ThreadLocal对应的数组下标，也就是空间换时间，实现代码：[FastThreadLocal.java](https://github.com/netty/netty/blob/4.1/common/src/main/java/io/netty/util/concurrent/FastThreadLocal.java)，当然，在Android中对同个线程的ThreadLocal哈希表频繁操作这种场景可能并不多见。