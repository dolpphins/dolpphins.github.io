---
title: OkHttp必须弄清楚的几个原理性问题
date: 2020-01-11 09:39:51
tags:
---

### 前言

okhttp是目前很火的网络请求框架，Android4.4开始HttpURLConnection的底层就是采用okhttp实现的，其Github地址：[https://github.com/square/okhttp](https://github.com/square/okhttp)

来自官方说明：

>   OkHttp is an HTTP client that’s efficient by default:
> 
>   * HTTP/2 support allows all requests to the same host to share a socket.
>   * Connection pooling reduces request latency (if HTTP/2 isn’t available).
>   * Transparent GZIP shrinks download sizes.
>   * Response caching avoids the network completely for repeat requests.

总结一下，OkHttp支持http2，当然需要你请求的服务端支持才行，针对http1.x，OkHttp采用了连接池降低网络延迟，内部实现gzip透明传输，使用者无需关注，支持http协议上的缓存用于避免重复网络请求。

### 使用方法

1. 引入依赖

		implementation 'com.squareup.okhttp3:okhttp:3.14.4'

2. 请求网络

		OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url("http://mtancode.com/").build();
        // 同步方式
        Response response = okHttpClient.newCall(request).execute();
        // 异步方式
        okHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.i(TAG, "onFailure");
                e.printStackTrace();
            }

            @Override
            public void onResponse(Call call, Response response) {
                try {
                    Log.i(TAG, response.body().string());
                } catch (Throwable t) {
                    t.printStackTrace();
                }
            }
        });

可以看到，使用起来非常简单，而且支持同步和异步两种方式请求网络。这里需要注意一下，回调的线程并不是UI线程。

### 主流程分析

同步和异步只是使用方式不同，但其原理都是一样的，最终会走到相同的逻辑，因此这里就直接从异步方式开始分析了，newCall方法会返回一个RealCall对象，看其enqueue方法：

	@Override public void enqueue(Callback responseCallback) {
	    synchronized (this) {
	      if (executed) throw new IllegalStateException("Already Executed");
	      executed = true;
	    }
	    transmitter.callStart();
	    client.dispatcher().enqueue(new AsyncCall(responseCallback));
	}

这里有个Dispatcher，顾名思义它就是专门分发和执行请求的，看它的enqueue方法：

	void enqueue(AsyncCall call) {
	    synchronized (this) {
	      readyAsyncCalls.add(call);
	
	      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
	      // the same host.
	      if (!call.get().forWebSocket) {
	        AsyncCall existingCall = findExistingCallWithHost(call.host());
	        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
	      }
	    }
	    promoteAndExecute();
	  }

把call添加到readyAsyncCalls列表中，看promoteAndExecute方法：

  	private boolean promoteAndExecute() {
    	assert (!Thread.holdsLock(this));

	    List<AsyncCall> executableCalls = new ArrayList<>();
	    boolean isRunning;
	    synchronized (this) {
	      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
	        AsyncCall asyncCall = i.next();
	
	        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
	        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.
	
	        i.remove();
	        asyncCall.callsPerHost().incrementAndGet();
	        executableCalls.add(asyncCall);
	        runningAsyncCalls.add(asyncCall);
	      }
	      isRunning = runningCallsCount() > 0;
	    }
	
	    for (int i = 0, size = executableCalls.size(); i < size; i++) {
	      AsyncCall asyncCall = executableCalls.get(i);
	      asyncCall.executeOn(executorService());
	    }
	
	    return isRunning;
  	}

把call搬到runningAsyncCalls中，遍历列表，对每个call调用executeOn方法：

    void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        transmitter.noMoreExchanges(ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }

看RealCall的execute方法：

	@Override protected void execute() {
      boolean signalledCallback = false;
      transmitter.timeoutEnter();
      try {
        Response response = getResponseWithInterceptorChain();
	    responseCallback.onResponse(RealCall.this, response);
	......
	}

来到getResponseWithInterceptorChain方法，该方法内部会执行所有具体的处理逻辑，执行结束后，返回一个最终的response，然后回调给外部传入的callback，看看getResponseWithInterceptorChain方法：

	Response getResponseWithInterceptorChain() throws IOException {
	    // Build a full stack of interceptors.
	    List<Interceptor> interceptors = new ArrayList<>();
	    interceptors.addAll(client.interceptors());
	    interceptors.add(new RetryAndFollowUpInterceptor(client));
	    interceptors.add(new BridgeInterceptor(client.cookieJar()));
	    interceptors.add(new CacheInterceptor(client.internalCache()));
	    interceptors.add(new ConnectInterceptor(client));
	    if (!forWebSocket) {
	      interceptors.addAll(client.networkInterceptors());
	    }
	    interceptors.add(new CallServerInterceptor(forWebSocket));
	
	    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
	        originalRequest, this, client.connectTimeoutMillis(),
	        client.readTimeoutMillis(), client.writeTimeoutMillis());
	
	    boolean calledNoMoreExchanges = false;
	    try {
	      Response response = chain.proceed(originalRequest);
	      if (transmitter.isCanceled()) {
	        closeQuietly(response);
	        throw new IOException("Canceled");
	      }
	      return response;
	    } catch (IOException e) {
	      calledNoMoreExchanges = true;
	      throw transmitter.noMoreExchanges(e);
	    } finally {
	      if (!calledNoMoreExchanges) {
	        transmitter.noMoreExchanges(null);
	      }
	    }
	  }

可以看到，这里添加了一系列的拦截器，构成拦截器链，请求会沿着这条链依次调用其intercept方法，每个拦截器都做自己该做的工作，最终完成请求，返回最终的response对象。

简单说下链式调用的实现方法：创建一个RealInterceptorChain，传入所有的interceptors，和当前index（从0开始），然后调用RealInterceptorChain的process方法，该方法里，获取到对应的interceptor，然后调用intercept方法，而在intercept方法中，会执行具体的处理逻辑，然后创建一个RealInterceptorChain，传入所有的interceptors，和当前index+1，继续调用RealInterceptorChain的process方法，如此重复直到index超过interceptors个数为止。其实这种实现方式跟Task实现链式调用很类似，整个调用过程会创建一系列的中间对象。

继续回到okhttp，这里其实是一种责任链设计模式，它的优点有：

* 可以降低逻辑的耦合，相互独立的逻辑写到自己的拦截器中，也无需关注其它拦截器所做的事情。

* 扩展性强，可以添加新的拦截器。

当然它也有缺点：

* 因为调用链路长，而且存在嵌套，遇到问题排查其它比较麻烦。

对于OkHttp，我们可以添加自己的拦截器：

	OkHttpClient.Builder builder = new OkHttpClient().newBuilder();
	builder.addInterceptor(new Interceptor() {
	    @Override
	    public Response intercept(Chain chain) throws IOException {
	        // TODO 自定义逻辑
	        
	        return chain.proceed(chain.request());
	    }
	});

来到这里，OkHttp的主流程就分析完了，至于具体的缓存逻辑，连接池逻辑，网络请求这些，都是在对应的拦截器里面实现的，下面对这些拦截器逐个进行分析。

### 缓存机制

代码在CacheInterceptor类中，实现HTTP协议的缓存机制，OkHttp默认并不没有开启缓存，要自己传入一个Cache对象。

先了解下HTTP协议的缓存机制：

首先缓存分为三种：过期时间缓存、第一差异缓存和第二差异缓存，而且在优先级上，过期时间缓存 > 第一差异缓存 > 第二差异缓存。

过期时间缓存，就是通过HTTP响应头部的字段控制：

* expires：响应字段，绝对过期时间，HTTP1.0。

* Cache-Control：响应字段，相对过期时间，HTTP1.1。注意如果值为no-cache，表示跳过过期时间缓存逻辑，值为no-store表示跳过过期时间缓存逻辑和差异缓存逻辑，也就是不使用缓存数据。

当客户端请求时，发现缓存未过期，就直接返回缓存数据了，不请求网络，否则，执行第一差异缓存逻辑：

* If-None-Match：请求字段，值为ETag。

* ETag：响应字段，服务端会根据内容生成唯一的字符串。

如果服务端发现If-None-Match的值和当前ETag一样，就说明数据内容没有变化，就返回304，否则，执行第二差异缓存逻辑：

* If-Modified-Since：请求字段，客户端告诉服务端本地缓存的资源的上次修改时间。

* Last-Modified：响应字段，服务端告诉客户端资源的最后修改时间。

如果服务端发现If-Modified-Since的值就是资源的最后修改时间，就说明数据内容没有变化，就返回304，否则，返回所有资源数据给客户端，响应码为200。

回到OkHttp，CacheInterceptor拦截器处理的逻辑，其实就是上面所说的HTTP缓存逻辑，注意到OkHttp提供了一个现成的缓存类Cache，它采用DiskLruCache实现缓存策略，至于缓存的位置和大小，需要你自己指定。

这里其实会有个问题，上面的缓存都是依赖HTTP协议本身的缓存机制的，如果我们请求的服务器不支持这套缓存机制，或者需要实现更灵活的缓存管理，直接使用上面这套缓存机制就可能不太可行了，这时我们可以自己新增拦截器，自行实现缓存的管理。

### 连接池

连接池的逻辑在ConnectInterceptor拦截器中处理，看intercept方法：

    @Override public Response intercept(Chain chain) throws IOException {
	    RealInterceptorChain realChain = (RealInterceptorChain) chain;
	    Request request = realChain.request();
	    Transmitter transmitter = realChain.transmitter();
	
	    // We need the network to satisfy this request. Possibly for validating a conditional GET.
	    boolean doExtensiveHealthChecks = !request.method().equals("GET");
	    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
	
	    return realChain.proceed(request, transmitter, exchange);
    }

关键代码就是调用了Transmitter的newExchange方法，最终会得到一个Exchange对象，该对象表示一条连接，用于后面实现请求和读取响应数据，为了避免陷入代码中无法自拔，这里就不一步一步跟踪newExchange方法了，它最后会调用ExchangeFinder的findConnection的方法，这个方法就是在连接池中寻找可复用的连接，当然如果没找到，就创建一个新的连接，OkHttp对连接池的管理是在RealConnectionPool类中：

	public final class RealConnectionPool {
	  /**
	   * Background threads are used to cleanup expired connections. There will be at most a single
	   * thread running per connection pool. The thread pool executor permits the pool itself to be
	   * garbage collected.
	   */
	  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
	      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
	      new SynchronousQueue<>(), Util.threadFactory("OkHttp ConnectionPool", true));
	
	  /** The maximum number of idle connections for each address. */
	  private final int maxIdleConnections;
	  private final long keepAliveDurationNs;
	  private final Runnable cleanupRunnable = () -> {
	    while (true) {
	      long waitNanos = cleanup(System.nanoTime());
	      if (waitNanos == -1) return;
	      if (waitNanos > 0) {
	        long waitMillis = waitNanos / 1000000L;
	        waitNanos -= (waitMillis * 1000000L);
	        synchronized (RealConnectionPool.this) {
	          try {
	            RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
	          } catch (InterruptedException ignored) {
	          }
	        }
	      }
	    }
	  };
	
	  private final Deque<RealConnection> connections = new ArrayDeque<>();
	  ......
	}

主要关注几个重要的成员变量，maxIdleConnections表示连接池的最大缓存连接数，这里外部传入了5，也就是最多缓存5个连接，缓存的连接都被放到connections中，而keepAliveDurationNs表示连接的缓存时长，这里为5分钟，我们还看到这里还有个executor，它就是用来清理过期连接。


### 数据传输

在CallServerInterceptor拦截器中处理，采用okio实现，http请求和读取响应最终是在Http1ExchangeCodec或Http2ExchangeCodec中实现的。

### 透明gzip压缩

在BridgeInterceptor拦截器中处理，看一下intercept方法：

	  @Override public Response intercept(Chain chain) throws IOException {
	    Request userRequest = chain.request();
	    Request.Builder requestBuilder = userRequest.newBuilder();
	
	    RequestBody body = userRequest.body();
		......
	
	    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
	    // the transfer stream.
	    boolean transparentGzip = false;
	    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
	      transparentGzip = true;
	      requestBuilder.header("Accept-Encoding", "gzip");
	    }
	
	    ......
	
	    if (transparentGzip
	        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
	        && HttpHeaders.hasBody(networkResponse)) {
	      GzipSource responseBody = new GzipSource(networkResponse.body().source());
	      Headers strippedHeaders = networkResponse.headers().newBuilder()
	          .removeAll("Content-Encoding")
	          .removeAll("Content-Length")
	          .build();
	      responseBuilder.headers(strippedHeaders);
	      String contentType = networkResponse.header("Content-Type");
	      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
	    }
	
	    return responseBuilder.build();
	  }

可以看到，OkHttp默认会为我们加上gzip头部字段，如果服务端支持的话，就会返回gzip压缩后的数据，这样就可以缩短传输时间和减少传输数据大小，接收到gzip压缩后的数据后，Okhttp会自动帮我们解压缩，所以这一切对使用者来说都是透明的，无需关注，当然如果我们自己明确指定了用gzip压缩，解压缩的事情就需要我们自己来做了。

### 支持HTTP2

OkHttp2支持HTTP2协议，当然如果服务端不支持就没办法了，针对HTTP2的相关类都在okhttp3.internal.http2包下，有兴趣可以自行查看源码。

关于HTTP2的优点，主要有：

* 多路复用：就是针对同个域名的请求，都可以在同一条连接中并行进行，而且头部和数据都进行了二进制封装。

* 二进制分帧：传输都是基于字节流进行的，而不是文本，二进制分帧层处于应用层和传输层之间。

* 头部压缩：HTTP1.x每次请求都会携带完整的头部字段，所以可能会出现重复传输，因此HTTP2采用HPACK对其进行压缩优化，可以节省不少的传输流量。

* 服务端推送：服务端可以主动推送数据给客户端。

### 参考文章

[解密HTTP/2与HTTP/3 的新特性](https://juejin.im/post/5d9abde7e51d4578110dc77f)
