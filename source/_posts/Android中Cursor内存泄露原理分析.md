---
title: Android中Cursor内存泄露原理分析
date: 2019-12-01 12:53:22
tags:
---

我们知道，Cursor使用结束后，如果没有调用Cursor的close方法，就可能会导致内存泄露，但其原因是什么呢？究竟是什么对象泄露了？下面通过源码分析其原因。

## SQLiteDatabase的query过程

1. SQLiteDatabase执行查询操作很简单，直接调用query方法：

	    CustomSQLiteDatabase helper = new CustomSQLiteDatabase(this);
	    SQLiteDatabase db = helper.getWritableDatabase();
	    Cursor cursor = db.query("COMPANY", null, null, null, null, null, null);

2. 内部会创建一个SQLiteDirectCursorDriver对象，然后调用其query方法：

	    public Cursor query(CursorFactory factory, String[] selectionArgs) {
	        final SQLiteQuery query = new SQLiteQuery(mDatabase, mSql, mCancellationSignal);
	        final Cursor cursor;
	        try {
				// 建立index->列名映射，最终放在Object数组中
	            query.bindAllArgsAsStrings(selectionArgs);
	
	            if (factory == null) {
	                cursor = new SQLiteCursor(this, mEditTable, query);
	            } else {
	                cursor = factory.newCursor(mDatabase, this, mEditTable, query);
	            }
	        } catch (RuntimeException ex) {
	            query.close();
	            throw ex;
	        }
	
	        mQuery = query;
	        return cursor;
	    }

	一般我们不会指定CursorFactory，所以factory为null，这种情况下创建的是一个SQLiteCursor对象，也就是外部的query方法返回的其实就是这个SQLiteCursor对象。

    另外注意到这里调用query方法时，只是做了一些初始化操作，简单返回一个SQLiteCursor对象而且，并不会真的去执行数据库查询，因此这个query执行起来不耗时。

    接着我们拿到这个SQLiteCursor对象后，调用moveToPosition(0)方法把游标移动到第一条记录，这个操作就比较耗时了，SQLiteCursor继承自AbstractWindowedCursor，看一下AbstractWindowedCursor的moveToPosition方法：

	    @Override
	    public final boolean moveToPosition(int position) {
	        // Make sure position isn't past the end of the cursor
	        final int count = getCount();
	        if (position >= count) {
	            mPos = count;
	            return false;
	        }
	
	        // Make sure position isn't before the beginning of the cursor
	        if (position < 0) {
	            mPos = -1;
	            return false;
	        }
	
	        // Check for no-op moves, and skip the rest of the work for them
	        if (position == mPos) {
	            return true;
	        }
	
	        boolean result = onMove(mPos, position);
	        if (result == false) {
	            mPos = -1;
	        } else {
	            mPos = position;
	        }
	
	        return result;
	    }

    这里的getCount方法和onMove方法，都可能会触发执行fillWindow方法，而fillWindow方法就是真正去查询数据库，然后把数据填充到共享内存中。对于这块共享内存，默认大小为2MB，只有当真正查询数据库时才会被创建，然后填充查询结果数据，假如结果数据过大填充不下，就只填充一部分，如果下次读取到填充之外的数据，就需要重新执行数据库查询，清空原有共享内存缓存数据，然后把新数据填充到共享内存中。

3. 继续看fillWindow方法：

	    private void fillWindow(int requiredPos) {
	        clearOrCreateWindow(getDatabase().getPath());
	        try {
	            Preconditions.checkArgumentNonnegative(requiredPos,
	                    "requiredPos cannot be negative, but was " + requiredPos);
	
	            if (mCount == NO_COUNT) {
	                mCount = mQuery.fillWindow(mWindow, requiredPos, requiredPos, true);
	                mCursorWindowCapacity = mWindow.getNumRows();
	                if (Log.isLoggable(TAG, Log.DEBUG)) {
	                    Log.d(TAG, "received count(*) from native_fill_window: " + mCount);
	                }
	            } else {
	                int startPos = mFillWindowForwardOnly ? requiredPos : DatabaseUtils
	                        .cursorPickFillWindowStartPosition(requiredPos, mCursorWindowCapacity);
	                mQuery.fillWindow(mWindow, startPos, requiredPos, false);
	            }
	        } catch (RuntimeException ex) {
	            // Close the cursor window if the query failed and therefore will
	            // not produce any results.  This helps to avoid accidentally leaking
	            // the cursor window if the client does not correctly handle exceptions
	            // and fails to close the cursor.
	            closeWindow();
	            throw ex;
	        }
	    }

	一开始就调用clearOrCreateWindow方法，如果共享内存已经存在就清空，没有就创建，这里其实就是创建一个CursorWindow对象，当然它只是一个壳而已，最终对共享内存的操作都是通过native层实现的。

	共享内存准备好之后，就可以调用mQuery的fillWindow方法查询数据库填充数据了，mQuery的类型为SQLiteQuery，fillWindow方法会连接到数据库查询数据，然后填充到共享内存中。


**通过上面的分析，Android采用CursorWindow共享内存缓存数据库查询结果数据，优点有：**

1. 能够实现跨进程传输数据，其它进程可以读取CursorWindow共享内存的数据。

2. 缓存数据，不用每次都查询数据库

3. 能够根据position快速定位到某一条记录

**当然这种做法也存在缺点：**

1. 中间读写CursorWindow有一定的性能开销

2. 遍历过程如果发生Cursor重新查询填充，也会消耗不少时间。

因此微信的[wcdb](https://github.com/Tencent/wcdb "https://github.com/Tencent/wcdb")就对Cursor做了优化，直接把CursorWindow共享内存去掉了，自己实现了一个SQLiteDirectCursor，把缓存做到Java层上，当然这个SQLiteDirectCursor就不能实现跨进程共享数据。这种基于特定场景进行改造优化，是常用的提升性能方法。

来到这里，我们可以猜测，没有调用close方法时这块共享内存就没有被释放，也就出现了内存泄露，下面分析close方法看看它是如何释放这块共享内存的。

## Cursor的close方法原理

分析之前，先理清SQLiteCursor的继承层次：SQLiteCursor --> AbstractWindowedCursor --> AbstractCursor --- CrossProcessCursor，

也就是，SQLiteCursor继承自AbstractWindowedCursor，AbstractWindowedCursor继承自AbstractCursor，而AbstractCursor又实现了CrossProcessCursor接口。

下面分析close方法调用流程：

1. 上面我们说到，query返回的其实是一个SQLiteCursor对象，看下它的close方法：

	    @Override
	    public void close() {
	        super.close();
	        synchronized (this) {
	            mQuery.close();
	            mDriver.cursorClosed();
	        }
	    }

2. AbstractCursor的close方法：

	    @Override
	    public void close() {
	        mClosed = true;
	        mContentObservable.unregisterAll();
	        onDeactivateOrClose();
	    }

3. AbstractWindowedCursor的onDeactivateOrClose方法：

	    @Override
	    protected void onDeactivateOrClose() {
	        super.onDeactivateOrClose();
	        closeWindow();
	    }

4. AbstractWindowedCursor的closeWindow方法：

	    protected void closeWindow() {
	        if (mWindow != null) {
	            mWindow.close();
	            mWindow = null;
	        }
	    }

	来到这里，就可以很清楚地看到，没有调用Cursor的close方法，CursorWindow的close方法也就不会被调用，那么这个CursorWindow共享内存也就发生泄漏了。

	CursorWindow的close方法，内部会维护一个引用计数，每次close一次，就减1，如果减到为0，就调用dispose方法，这个方法就是真正去释放资源，可以看看其实现代码：

	    private void dispose() {
	        if (mCloseGuard != null) {
	            mCloseGuard.close();
	        }
	        if (mWindowPtr != 0) {
	            recordClosingOfWindow(mWindowPtr);
	            nativeDispose(mWindowPtr);
	            mWindowPtr = 0;
	        }
    	}

	最终通过native层关闭共享内存，释放相关资源。

## 总结

如果没有调用Cursor的close方法，就可能会导致本次query对应的共享内存发生内存泄露，因此每次使用完Cursor都必须调用其close方法，建议放到finally语句块里。