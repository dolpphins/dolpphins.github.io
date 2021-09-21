---
title: SharedPreferences必须弄清楚的几个问题
date: 2020-04-11 13:16:39
tags:
---

> 先推荐一下官方的Android源码查看网站：[Android Code Search](https://cs.android.com/)，功能很强大，用起来很方便。

### 原理分析

之前写过一篇文章：[深入理解Android中的SharedPreferences](https://blog.csdn.net/u012619640/article/details/50940074)，现在抛开源码，总结成几个关键点：

1. 只要file name相同，拿到的就是同一个SharedPreferencesImpl对象，内部有缓存机制，首次获取才会创建对象。

2. 在SharedPreferencesImpl构造方法中，会开启子线程把对应的文件key-value全部加载进内存，加载结束后，mLoaded被设置为true。

3. 调用getXXX方法时，会阻塞等待直到mLoaded为true，也就是getXXX方法是有可能阻塞UI线程的，另外，调用contains和
edit等方法也是。

4. 写数据时，会先拿到一个EditorImpl对象，然后putXXX，这时只是把数据写入到内存中，最后调用commit或者apply方法，才会真正写入文件。

5. 不管是commit还是apply方法，第一步都是调用commitToMemory方法生成一个MemoryCommitResult对象，注意这里会先处理clear旧的key-value，再处理新添加的key-value，另外value为this或者null都表示需要被remove掉。

6. 调用commit方法，就会同步执行写入文件的操作，该方法是耗时操作，不能在主线程中调用，该方法最后会返回成功或失败结果。

7. 调用apply方法，就会把任务放到QueuedWork的队列中，然后在HandlerThread中执行，然后apply方法会立即返回。但如果是Android8.0之前，这里就是放到QueuedWork的一个单线程中执行了。

8. 最后是写入文件，会先把原有的文件命名为bak备份文件，然后创建新的文件全量写入，写入成功后，把bak备份文件删除掉。

### apply引起的ANR

从上面的分析来看，apply方法调用后会立即返回，看起来很安全，不会阻塞主线程，但再仔细看下apply的代码：

	@Override
    public void apply() {
        final long startTime = System.currentTimeMillis();

        final MemoryCommitResult mcr = commitToMemory();
        final Runnable awaitCommit = new Runnable() {
                @Override
                public void run() {
                    try {
                        mcr.writtenToDiskLatch.await();
                    } catch (InterruptedException ignored) {
                    }

                    if (DEBUG && mcr.wasWritten) {
                        Log.d(TAG, mFile.getName() + ":" + mcr.memoryStateGeneration
                                + " applied after " + (System.currentTimeMillis() - startTime)
                                + " ms");
                    }
                }
            };

        QueuedWork.addFinisher(awaitCommit);

        Runnable postWriteRunnable = new Runnable() {
                @Override
                public void run() {
                    awaitCommit.run();
                    QueuedWork.removeFinisher(awaitCommit);
                }
            };

        SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

        // Okay to notify the listeners before it's hit disk
        // because the listeners should always get the same
        // SharedPreferences instance back, which has the
        // changes reflected in memory.
        notifyListeners(mcr);
    }

这里会调用addFinisher把awaitCommit添加进QueuedWork的队列，awaitCommit的任务就是等apply的写文件任务执行完成，没有完成的话就等，而这个awaitCommit的run方法是在QueueWork的waitToFinish方法中调用的。

至于这个waitToFinish方法，它的调用时机就比较多了，比如Activity的onStop回调之后，Service停止之后等等，这些调用都是在主线程中进行的，当触发到这些时机时，awaitCommit的run方法就会被执行起来了，如果这时apply的写文件任务还没有完成，run方法就会阻塞住，就可能会导致ANR。

这个问题只会出现在Android8.0之前，看下7.1.1版本waitToFinish方法的代码：

    public static void waitToFinish() {
        Runnable toFinish;
        while ((toFinish = sPendingWorkFinishers.poll()) != null) {
            toFinish.run();
        }
    }
    
可以看到如果SharedPreferences的apply写入文件任务一直没有执行结束，这里什么事也不做，就一直等待着，最终就会导致ANR。

在Android8.0，Google进行了优化，看下8.0版本waitToFinish方法的代码：

	public static void waitToFinish() {
        long startTime = System.currentTimeMillis();
        boolean hadMessages = false;

        Handler handler = getHandler();

        synchronized (sLock) {
            if (handler.hasMessages(QueuedWorkHandler.MSG_RUN)) {
                // Delayed work will be processed at processPendingWork() below
                handler.removeMessages(QueuedWorkHandler.MSG_RUN);

                if (DEBUG) {
                    hadMessages = true;
                    Log.d(LOG_TAG, "waiting");
                }
            }

            // We should not delay any work as this might delay the finishers
            sCanDelay = false;
        }

        StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
        try {
            processPendingWork();
        } finally {
            StrictMode.setThreadPolicy(oldPolicy);
        }

        try {
            while (true) {
                Runnable finisher;

                synchronized (sLock) {
                    finisher = sFinishers.poll();
                }

                if (finisher == null) {
                    break;
                }

                finisher.run();
            }
        } finally {
            sCanDelay = true;
        }

        synchronized (sLock) {
            long waitTime = System.currentTimeMillis() - startTime;

            if (waitTime > 0 || hadMessages) {
                mWaitTimes.add(Long.valueOf(waitTime).intValue());
                mNumWaits++;

                if (DEBUG || mNumWaits % 1024 == 0 || waitTime > MAX_WAIT_TIME_MILLIS) {
                    mWaitTimes.log(LOG_TAG, "waited: ");
                }
            }
        }
    }
    
这里会先调用processPendingWork方法处理任务，处理完成后，再调用所有Finisher的run方法，这时就不会出现阻塞了，注意这个processPendingWork方法是直接在主线程中处理任务了，如果任务执行时间太久，还是会阻塞主线程的。

根据上面的分析，Android8.0以下的是可能会出现ANR的，解决方法比较简单，直接反射把sFinishers清空就可以了，清空的时机就是waitToFinish方法的调用时机之前。

### 多进程方案

* ContentProvider

	使用ContentProvider进行跨进程通信，把其它进程的读写sp都统一到某个进程上。
	
* MMKV
	
	使用共享内存进行读写sp。

### 使用建议

1. 在主线程中调用apply方法，在子线程中直接调用commit方法。

2. 每一个SharedPreferences文件都不能存储太多东西，不然在首次加载进内存时会比较久，在主线程中调用getXXX等接口可能会出现ANR，同时占用的内存也会比较多。另外，每次修改写入都是全量写入文件，文件内容多的话写入就会更慢。

3. 尽量在调用完所有putXXX方法后，再统一进行提交。

