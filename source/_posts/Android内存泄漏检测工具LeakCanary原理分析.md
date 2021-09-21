---
title: Android内存泄漏检测工具LeakCanary原理分析
date: 2019-12-01 11:11:54
tags:
---

## LeakCanary是什么

LeakCanary是square推出的用于检测Android内存泄漏的开源工具，使用起来十分简单，接入成本低，项目地址：[https://github.com/square/leakcanary](https://github.com/square/leakcanary "LeakCanary Github链接")。

## 使用方法

	// module的build.gradle添加依赖
	implementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'

	public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        LeakCanary.install(this);
    }

然后运行程序，如果LeakCanary检测到可能存在内存泄露，就会发送一条通知，点击就可以看到内存泄露的引用链，当然你还必须自己去review代码，判定是否真的发生内存泄露，因为LeakCanary是有可能误判的。另外LeakCanary只会默认帮你监控Activity泄露，如果要监控某个对象是否发生内存泄露，就要自己手动去watch该对象。

这里还有一个问题需要注意，正式包是不能把LeakCanary打到包里面的，我们只是在开发包和测试包中使用，那这行LeakCanary.install(this)必须手动删掉，或者加一些测试包的判断逻辑，如果是自己手动watch其它对象，可能代码更多，处理起来不是很方便，因此square又提供了一个leakcanary-no-op的库，其实里面就是空实现，用于编译正式包，最终的依赖可以这样子写：

	// module的build.gradle添加依赖
	testImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
	debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.6.3'
	releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.6.3'
	
leakcanary-android-no-op的RefWatcher代码：


	/**
	 * No-op implementation of {@link RefWatcher} for release builds. Please use {@link
	 * RefWatcher#DISABLED}.
	 */
	public final class RefWatcher {
	
	  @NonNull public static final RefWatcher DISABLED = new RefWatcher();
	
	  private RefWatcher() {
	  }
	
	  public void watch(@NonNull Object watchedReference) {
	  }
	
	  public void watch(@NonNull Object watchedReference, @NonNull String referenceName) {
	  }
	}

## 原理分析

1. LeakCanary的install方法：

	    public static @NonNull RefWatcher install(@NonNull Application application) {
	    	return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
	       		.excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
	        	.buildAndInstall();
	  	}

2. 这里refWatcher(application)会返回一个AndroidRefWatcherBuilder对象，listenerServiceClass方法指定处理显示内存泄露通知的服务类，excludedRefs方法指定白名单，这个白名单可以自己配置，避免LeakCanary误判。接下来看buildAndInstall方法：

	    public @NonNull RefWatcher buildAndInstall() {
		    if (LeakCanaryInternals.installedRefWatcher != null) {
		      throw new UnsupportedOperationException("buildAndInstall() should only be called once.");
		    }
		    RefWatcher refWatcher = build();
		    if (refWatcher != DISABLED) {
		      if (enableDisplayLeakActivity) {
		        LeakCanaryInternals.setEnabledAsync(context, DisplayLeakActivity.class, true);
		      }
		      if (watchActivities) {
		        ActivityRefWatcher.install(context, refWatcher);
		      }
		      if (watchFragments) {
		        FragmentRefWatcher.Helper.install(context, refWatcher);
		      }
		    }
		    LeakCanaryInternals.installedRefWatcher = refWatcher;
		    return refWatcher;
		}

3. 调用ActivityRefWatcher.install对Activity内存泄露进行检测：

	    public static void install(@NonNull Context context, @NonNull RefWatcher refWatcher) {
		    Application application = (Application) context.getApplicationContext();
		    ActivityRefWatcher activityRefWatcher = new ActivityRefWatcher(application, refWatcher);
		
		    application.registerActivityLifecycleCallbacks(activityRefWatcher.lifecycleCallbacks);
		}

4. registerActivityLifecycleCallbacks是系统API，应用内任何一个Activity生命周期发生改变都会回调相应的方法，包括onDestroy，看ActivityRefWatcher的lifecycleCallbacks回调：

	    private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
		    new ActivityLifecycleCallbacksAdapter() {
		      @Override public void onActivityDestroyed(Activity activity) {
		        refWatcher.watch(activity);
		      }
	    };

5. 这里回调传过来的activity就是回调了onDestroy的Activity对象，理论上接下来它应该被回收，有了这个对象引用后，调用了RefWatcher的watch方法：

	    public void watch(Object watchedReference, String referenceName) {
		    if (this == DISABLED) {
		      return;
		    }
		    checkNotNull(watchedReference, "watchedReference");
		    checkNotNull(referenceName, "referenceName");
		    final long watchStartNanoTime = System.nanoTime();
		    String key = UUID.randomUUID().toString();
		    retainedKeys.add(key);
		    final KeyedWeakReference reference =
		        new KeyedWeakReference(watchedReference, key, referenceName, queue);
		
		    ensureGoneAsync(watchStartNanoTime, reference);
		}

6. 首先生成一个唯一的key，然后放到retainedKeys集合中，然后再创建一个WeakReference引用到Activity对象上，指定queue为其ReferenceQueue，最后调用ensureGoneAsync方法判断该Activity对象是否存在内存泄露：

	    private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
		    watchExecutor.execute(new Retryable() {
		      @Override public Retryable.Result run() {
		        return ensureGone(reference, watchStartNanoTime);
		      }
		    });
		}

7. 这个watchExecutor的execute方法，内部最后会在主线程中添加一个Idler消息，也就是ensureGone方法，最终是运行在主线程上的，而且只有等主线程空闲了没消息处理了，才会去执行它。看看ensureGone方法：

	    Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
		    long gcStartNanoTime = System.nanoTime();
		    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
		
		    removeWeaklyReachableReferences();
		
		    if (debuggerControl.isDebuggerAttached()) {
		      // The debugger can create false leaks.
		      return RETRY;
		    }
		    if (gone(reference)) {
		      return DONE;
		    }
		    gcTrigger.runGc();
		    removeWeaklyReachableReferences();
		    if (!gone(reference)) {
		      long startDumpHeap = System.nanoTime();
		      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
		
		      File heapDumpFile = heapDumper.dumpHeap();
		      if (heapDumpFile == RETRY_LATER) {
		        // Could not dump the heap.
		        return RETRY;
		      }
		      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
		
		      HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
		          .referenceName(reference.name)
		          .watchDurationMs(watchDurationMs)
		          .gcDurationMs(gcDurationMs)
		          .heapDumpDurationMs(heapDumpDurationMs)
		          .build();
		
		      heapdumpListener.analyze(heapDump);
		    }
		    return DONE;
		}

8. 这个ensureGone就是最关键的部分了，首先会调用removeWeaklyReachableReferences方法，在该方法中，如果判断到queue队列不为空，就说明垃圾回收器认为该Activity对象需要被回收了，那么就一切正常，没有发生内存泄露，就移除掉retainedKeys中该Activity对应的唯一key，接着gone方法就会判断到retainedKeys已经不存在该Activity对应的唯一key了，返回true，流程结束。

	但假如最开始判断到queue队列为空，就说明该Activity对象还没有被认为可以回收，这时先手动触发一下gc，最大可能保证垃圾回收器已经处理过了（判断为确实不需要回收，而不是因为某些原因根本就没去判断），然后再次调用removeWeaklyReachableReferences方法判断queue队列是否为空，如果为空，就LeakCanary断定该Activity发生内存泄露，通过android.os.Debug.dumpHprofData方法获取到当前堆信息，最后分析堆信息得到引用链。


## 改进LeakCanary

从上面原理分析，可以看到LeakCanary是存在一些问题，我们可以对其进行改进，改进措施主要有下面几种：

* 确保有gc

	LeakCanary会主动去调用gc接口，当最终虚拟机有没有真正gc就不知道了，如果没有，那么ReferenceQueue队列为空，就会判断为内存泄露，这样就可能误判，因此我们可以新增一个“哨兵”，只要它被回收，那么就说明虚拟机肯定进行过gc了，如果没有，就说明虚拟机没有进行过gc，代码如下：

		final WeakReference<Object> sentinelRef = new WeakReference<>(new Object());
        triggerGc();
        if (sentinelRef.get() != null) {
            // System ignored our gc request, we will retry later.
            MatrixLog.d(TAG, "system ignore our gc request, wait for next detection.");
            return Status.RETRY;
        }

	可以看到，实现起来很简单，new一个Object对象就可以，然后触发gc，最后判断这个Object对象是否被回收了。

* 多次判定

	假设有一种情况，LeakCanary在判定某个对象是否发生内存泄露时，该对象刚好被局部变量引用着，那这时就判定为没有内存泄露，但局部变量马上就释放引用了，这种情况LeakCanary就漏掉了，所以这里可以多判定几次，比如3次。

* 提醒去重

	如果某个对象被LeakCanary判定为内存泄露了，就会通知提示，如果多次判断，就会多次通知，体验不好，我们可以把泄露对象类名记录起来，只提醒一次。
	
微信开源的性能检测框架[Matrix](https://github.com/Tencent/matrix "https://github.com/Tencent/matrix")，就对LeakCanary进行了改进，改进方法基本跟上面所说的几点一样，具体细节可以自行分析其源码。

## 补充

1. 对于WeakReference的queue，当关联的对象只有被弱引用引用着的时候，该弱引用对象（注意不是关联的强引用对象）就会被进队列到queue中。

2. 在首次判断到queue为空时，会先手动触发gc，代码如下：

	    public interface GcTrigger {
		    GcTrigger DEFAULT = new GcTrigger() {
		        public void runGc() {
		            Runtime.getRuntime().gc();
		            this.enqueueReferences();
		            System.runFinalization();
		        }
		
		        private void enqueueReferences() {
		            try {
		                Thread.sleep(100L);
		            } catch (InterruptedException var2) {
		                throw new AssertionError();
		            }
		        }
		    };
		
		    void runGc();
		}

	这里会尽量让垃圾回收器执行垃圾回收，但不是一定执行。

3. dumpHprofData方法只能dump当前进程的内存信息，因此对于多进程应用，LeakCanary需要对每个进程都进行监控。

4. 分析内存信息得到引用链，是通过开源库[haha](https://github.com/square/haha "haha")实现的。