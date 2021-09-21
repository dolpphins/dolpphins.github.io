---
title: Android线程死锁检测方案设计
date: 2019-12-07 10:18:56
tags:
---

## 前言

在项目中，使用多线程是很常见的事情，但是如果处理不当，代码写的不好，就可能会导致线程死锁，对于死锁问题，从发现到定位问题都是比较困难的，如果是线上用户发生了线程死锁，那就是难上加难了，因此最好是项目本身有自己一套线程死锁检测机制，能够自动检测，自动上报，然后我们分析上报日志就可以了。

## WatchDog原理

在讨论如何设计一套完整的线程死锁检测方案之前，先了解一下Android系统中的WatchDog实现原理。WatchDog的本质是一条死锁检测线程，运行在SystemServer进程中，它的作用是不断地检测AMS，WMS等这些关键服务是否出现死锁，假如死锁了，就停止SystemServer进程，这时Zygote进程也会自杀，然后重启，也就是手机系统重启。

##### 接下来通过源码分析WatchDog的实现原理。

1. Zygote进程启动SystemServer进程后，会调用到SystemServer类的main静态方法：

	    /**
	     * The main entry point from zygote.
	     */
	    public static void main(String[] args) {
	        new SystemServer().run();
	    }

2. 直接看run方法：

		private void run() {	
			......
			// Start services.
	        try {
	            traceBeginAndSlog("StartServices");
	            startBootstrapServices();
	            startCoreServices();
	            startOtherServices();
	            SystemServerInitThreadPool.shutdown();
	        } catch (Throwable ex) {
	            Slog.e("System", "******************************************");
	            Slog.e("System", "************ Failure starting system services", ex);
	            throw ex;
	        } finally {
	            traceEnd();
	        }
			......
		}

3. startOtherServices方法会启动很多关键服务，包括AMS，WMS，WatchDog也是在这里面启动的：

	    private void startOtherServices() {
			......
			traceBeginAndSlog("StartWatchdog");
	        Watchdog.getInstance().start();
	        traceEnd();
		}

4. 可以看到，WatchDog被设计为单例，因为WatchDog继承自Thread类，也就是这里调用start方法，最终会开启一个检测线程，然后执行其run方法：

		@Override
	    public void run() {
	        boolean waitedHalf = false;
	        while (true) {
	            final List<HandlerChecker> blockedCheckers;
	            final String subject;
	            final boolean allowRestart;
	            int debuggerWasConnected = 0;
	            synchronized (this) {
	                long timeout = CHECK_INTERVAL;
	                // Make sure we (re)spin the checkers that have become idle within
	                // this wait-and-check interval
	                for (int i=0; i<mHandlerCheckers.size(); i++) {
	                    HandlerChecker hc = mHandlerCheckers.get(i);
	                    hc.scheduleCheckLocked();
	                }
	
	                if (debuggerWasConnected > 0) {
	                    debuggerWasConnected--;
	                }
	
	                // NOTE: We use uptimeMillis() here because we do not want to increment the time we
	                // wait while asleep. If the device is asleep then the thing that we are waiting
	                // to timeout on is asleep as well and won't have a chance to run, causing a false
	                // positive on when to kill things.
	                long start = SystemClock.uptimeMillis();
	                while (timeout > 0) {
	                    if (Debug.isDebuggerConnected()) {
	                        debuggerWasConnected = 2;
	                    }
	                    try {
	                        wait(timeout);
	                    } catch (InterruptedException e) {
	                        Log.wtf(TAG, e);
	                    }
	                    if (Debug.isDebuggerConnected()) {
	                        debuggerWasConnected = 2;
	                    }
	                    timeout = CHECK_INTERVAL - (SystemClock.uptimeMillis() - start);
	                }
	
	                boolean fdLimitTriggered = false;
	                if (mOpenFdMonitor != null) {
	                    fdLimitTriggered = mOpenFdMonitor.monitor();
	                }
	
	                if (!fdLimitTriggered) {
	                    final int waitState = evaluateCheckerCompletionLocked();
	                    if (waitState == COMPLETED) {
	                        // The monitors have returned; reset
	                        waitedHalf = false;
	                        continue;
	                    } else if (waitState == WAITING) {
	                        // still waiting but within their configured intervals; back off and recheck
	                        continue;
	                    } else if (waitState == WAITED_HALF) {
	                        if (!waitedHalf) {
	                            // We've waited half the deadlock-detection interval.  Pull a stack
	                            // trace and wait another half.
	                            ArrayList<Integer> pids = new ArrayList<Integer>();
	                            pids.add(Process.myPid());
	                            ActivityManagerService.dumpStackTraces(true, pids, null, null,
	                                getInterestingNativePids());
	                            waitedHalf = true;
	                        }
	                        continue;
	                    }
	
	                    // something is overdue!
	                    blockedCheckers = getBlockedCheckersLocked();
	                    subject = describeCheckersLocked(blockedCheckers);
	                } else {
	                    blockedCheckers = Collections.emptyList();
	                    subject = "Open FD high water mark reached";
	                }
	                allowRestart = mAllowRestart;
	            }
	
	            // If we got here, that means that the system is most likely hung.
	            // First collect stack traces from all threads of the system process.
	            // Then kill this process so that the system will restart.
	            EventLog.writeEvent(EventLogTags.WATCHDOG, subject);
	
	            ArrayList<Integer> pids = new ArrayList<>();
	            pids.add(Process.myPid());
	            if (mPhonePid > 0) pids.add(mPhonePid);
	            // Pass !waitedHalf so that just in case we somehow wind up here without having
	            // dumped the halfway stacks, we properly re-initialize the trace file.
	            final File stack = ActivityManagerService.dumpStackTraces(
	                    !waitedHalf, pids, null, null, getInterestingNativePids());
	
	            // Give some extra time to make sure the stack traces get written.
	            // The system's been hanging for a minute, another second or two won't hurt much.
	            SystemClock.sleep(2000);
	
	            // Trigger the kernel to dump all blocked threads, and backtraces on all CPUs to the kernel log
	            doSysRq('w');
	            doSysRq('l');
	
	            // Try to add the error to the dropbox, but assuming that the ActivityManager
	            // itself may be deadlocked.  (which has happened, causing this statement to
	            // deadlock and the watchdog as a whole to be ineffective)
	            Thread dropboxThread = new Thread("watchdogWriteToDropbox") {
	                    public void run() {
	                        mActivity.addErrorToDropBox(
	                                "watchdog", null, "system_server", null, null,
	                                subject, null, stack, null);
	                    }
	                };
	            dropboxThread.start();
	            try {
	                dropboxThread.join(2000);  // wait up to 2 seconds for it to return.
	            } catch (InterruptedException ignored) {}
	
	            IActivityController controller;
	            synchronized (this) {
	                controller = mController;
	            }
	            if (controller != null) {
	                Slog.i(TAG, "Reporting stuck state to activity controller");
	                try {
	                    Binder.setDumpDisabled("Service dumps disabled due to hung system process.");
	                    // 1 = keep waiting, -1 = kill system
	                    int res = controller.systemNotResponding(subject);
	                    if (res >= 0) {
	                        Slog.i(TAG, "Activity controller requested to coninue to wait");
	                        waitedHalf = false;
	                        continue;
	                    }
	                } catch (RemoteException e) {
	                }
	            }
	
	            // Only kill the process if the debugger is not attached.
	            if (Debug.isDebuggerConnected()) {
	                debuggerWasConnected = 2;
	            }
	            if (debuggerWasConnected >= 2) {
	                Slog.w(TAG, "Debugger connected: Watchdog is *not* killing the system process");
	            } else if (debuggerWasConnected > 0) {
	                Slog.w(TAG, "Debugger was connected: Watchdog is *not* killing the system process");
	            } else if (!allowRestart) {
	                Slog.w(TAG, "Restart not allowed: Watchdog is *not* killing the system process");
	            } else {
	                Slog.w(TAG, "*** WATCHDOG KILLING SYSTEM PROCESS: " + subject);
	                WatchdogDiagnostics.diagnoseCheckers(blockedCheckers);
	                Slog.w(TAG, "*** GOODBYE!");
	                Process.killProcess(Process.myPid());
	                System.exit(10);
	            }
	
	            waitedHalf = false;
	        }
	    }

	意料之中，这个run方法就是一个死循环，不断执行检测逻辑。至于具体的检测逻辑，可以分为三部分：

	(1) 检测是否发生死锁。

	(2) 如果发生死锁，就收集SystemServer所有线程信息。

	(3) 杀死SystemServer进程。

	下面分析这三部分的具体实现细节，先看WatchDog是如何检测有没有死锁的，这里涉及到几个变量，先了解一下：

		public class Watchdog extends Thread {
			......
			final ArrayList<HandlerChecker> mHandlerCheckers = new ArrayList<>();
		    final HandlerChecker mMonitorChecker;
			......
		}

	HandlerChecker也就是检测器，一个HandlerChecker对象用于检测一个线程，因为我们要检测多个线程，所以这里就是一个ArrayList列表。而HandlerChecker内部，会有一个Handler对象引用，它对应的Looper也就是检测线程的Looper，我们可以用这个Handler对象引用向检测线程Looper发消息，另外，HandlerChecker还有一个Monitor列表，这个就是被检测线程中检测对象列表，比如UI线程中的AMS，WMS这些对象，都会加到检测UI线程的HandlerChecker对象中的Monitor列表里面，当然AMS，WMS类他们都实现了Monitor接口。它们的关系如下：

	1个WatchDog --> n个HandlerChecker --> 对应n个检测线程

	1个HandlerChekcer --> n个Monitor --> 对应n个检测对象

	回到如何检测死锁的问题上面，这里会遍历HandlerChecker列表，对每个HandlerChecker对象，用对应线程的Handler对象post一个任务到消息队列头部，任务执行时会遍历Monitor列表，对每一个Monitor对象调用monitor方法，monitor方法的实现，就是具体类自己实现了，比如AMS的monitor方法：

	    /** In this method we try to acquire our lock to make sure that we have not deadlocked */
	    public void monitor() {
	        synchronized (this) { }
	    }

	WMS的monitor方法：

		// Called by the heartbeat to ensure locks are not held indefnitely (for deadlock detection).
	    @Override
	    public void monitor() {
	        synchronized (mWindowMap) { }
	    }

	可以看到，它们的检测逻辑都是看能不能正常获取锁，如果锁一直被别的线程占用着，这里就会一直等待直到拿到锁为止。注意AMS中使用的锁是this，WMS使用的锁是mWindowMap，只要能够正常获取这个锁，就说明服务没问题，处于正常状态，相反，不能获取这个锁，就说明锁因为未知原因被某个线程长期占用了，服务其它方法无法获取这个锁会导致无法被调用返回，也就是服务处于异常状态。

	其实这里阻塞除了一直在等待锁外，还有可能就是monitor方法根本没被调用，虽然monitor方法执行任务被放到Looper的消息队列最前面，但如果之前的消息阻塞了，就会导致monitor方法无法被执行到，这种情况也会被认为是死锁了。

	继续往下看，给所有线程发送所有monitor执行任务后，检测线程就开始等待了，这里等待30s，等待结束后，就开始查看所有monitor任务的执行状态，执行完的就属于正常，没执行或者还在执行，就说明有线程发送死锁了，这时开始收集所有线程信息：

		// If we got here, that means that the system is most likely hung.
	    // First collect stack traces from all threads of the system process.
	    // Then kill this process so that the system will restart.
	    EventLog.writeEvent(EventLogTags.WATCHDOG, subject);
	
	    ArrayList<Integer> pids = new ArrayList<>();
	    pids.add(Process.myPid());
	    if (mPhonePid > 0) pids.add(mPhonePid);
	    // Pass !waitedHalf so that just in case we somehow wind up here without having
	    // dumped the halfway stacks, we properly re-initialize the trace file.
	    final File stack = ActivityManagerService.dumpStackTraces(
	            !waitedHalf, pids, null, null, getInterestingNativePids());
	
	    // Give some extra time to make sure the stack traces get written.
	    // The system's been hanging for a minute, another second or two won't hurt much.
	    SystemClock.sleep(2000);

	收集到的线程信息，会被输出到/data/anr/traces.txt文件中。

	最后是WatchDog杀死SystemServer，这个比较简单，调用Process的killProcess方法实现。

		Slog.w(TAG, "*** WATCHDOG KILLING SYSTEM PROCESS: " + subject);
        WatchdogDiagnostics.diagnoseCheckers(blockedCheckers);
        Slog.w(TAG, "*** GOODBYE!");
        Process.killProcess(Process.myPid());
        System.exit(10);

##### 总结：WatchDog的核心就是检测死锁逻辑，开死循环线程检测所有线程是否阻塞住了，如果是就说明发生死锁了。

## 方案设计

设计线程死锁检测方案，主要考虑两个方面：检测逻辑和线程信息抓取。

* 检测逻辑

	* 方案一

		这个与WatchDog类似，开个死循环线程检测其它线程的Looper是否阻塞住。这里的阻塞，包括消息队列阻塞和无法获取关键对象锁两种。

	* 方案二

		死循环线程定时（比如5s）检查所有线程的状态，如果发现至少有两个线程处于BLOCKED状态的时间过长（比如3分钟），就认为有线程死锁了，这种方法对于ReentrantLock不适用，阻塞时获取到的线程状态为WAITING而不是BLOCKED。
		
		Java线程的六个状态说明参考：[Java Thread States and Life Cycle](https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html "https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html")

* 线程信息抓取

	因为是在App内抓取，相对比AMS，可能会有权限问题。我们可以直接抓线程的堆栈信息，但这样不会带有哪个线程正在持有哪个锁这个信息，不利于我们分析定位问题，而根据 [《手Q Android线程死锁监控与自动化分析实践》](https://cloud.tencent.com/developer/article/1064396 "手Q Android线程死锁监控与自动化分析实践") 这篇文章，它是把使用锁归类为三种，synchronized、wait/notify和ReentrantLock，然后前两种系统的/data/anr/traces.txt文件会有哪个线程持有哪个锁，哪个线程正在等待哪个锁这个信息，因此app只需要发送SIGQUIT信号给ANR进程触发生成traces.txt文件即可，而对于ReentrantLock，只能在代码里手动记录锁的线程使用信息。但这里看完有两个问题：

	1. 有些手机没有权限读取/data/anr/traces.txt这个文件，或者这个文件名被叫成traces-xxx，怎么获取到这个名称？
	
		这个暂时找不到解决方法，读取不到的话只有线程堆栈信息了。

	2. App怎么发送SIGQUIT信号给ANR进程？

		这个比较简单，android.os.Process有现成的接口：

			Process.sendSignal(Process.myPid(), Process.SIGNAL_QUIT);

	当然如果只通过调用栈信息大部分情况下也是可以分析出死锁的原因的，这时就不需要谁持有锁这个信息了。

