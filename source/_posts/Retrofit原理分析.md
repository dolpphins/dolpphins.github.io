---
title: Retrofit原理分析
date: 2021-03-27 22:09:29
tags:
---

### 一、前言

Retrofit是square开源的，用于封装网络请求方法和参数的第三方库，开发者使用注解进行网络请求配置，然后转由OkHttp请求网络。

### 二、基础用法

1. 定义接口：

		interface Animal {
		    @GET("/")
		    Call<ResponseBody> get();
		}

2. 创建Retrofit对象：

		Retrofit retrofit = new Retrofit.Builder()
		        .baseUrl("https://mtancode.com")
		        .build();

3. 调用接口方法：

		Animal animal = retrofit.create(Animal.class);
	    animal.get().enqueue(new Callback<ResponseBody>() {
	        @Override
	        public void onResponse(Call<ResponseBody> call,
	                               Response<ResponseBody> response) {
	            try {
	                Log.i("test", response.body().string());
	            } catch (Throwable t) {
	                t.printStackTrace();
	            }
	        }
	
	        @Override
	        public void onFailure(Call<ResponseBody> call, Throwable t) {
	            Log.i("test", "onFailure");
	        }
	    });

	请求结果在onResponse回调中获取到，该回调运行在UI线程中。

### 三、高级用法

1. 常用的注解还有：

	@Query：GET方式单个请求参数。

	@QueryMap：GET方式多个请求参数。

	@Path：url部分动态传入。

	@Url：url整体动态传入。

	@Field：POST方式单个请求参数。

	@FieldMap：POST方式多个请求参数。

	@Body：POST方式多个不同类型的请求参数。

2. addCallAdapterFactory

	改变接口返回类型，默认是retrofit2.Call，而其enqueue方法内部就是用okhttp实现网络请求的，如果要改成用其它请求方式（比如项目已有的网络请求框架，RxJava等），就可以自定义CallAdapter，然后接口返回自己的类型，再调用方法使用自己的请求方式，自定义CallAdapter需要三个类CustomCall，CustomCallAdapter和CustomCallAdapterFactory，如下：
		
		private static class CustomCall<T> {
		    private Call call;
		    CustomCall(Call call) {
		        this.call = call;
		    }
		    public void request() {
		        // TODO 自定义网络请求逻辑
		    }
		}
		
		private static class CustomCallAdapter implements CallAdapter<CustomCall, Object> {
		
		    @Override
		    public Type responseType() {
		        return ResponseBody.class;
		    }
		
		    @Override
		    public Object adapt(Call call) {
		        Log.i("test", call.toString());
		        return new CustomCall(call);
		    }
		}
		
		private static class CustomCallAdapterFactory extends CallAdapter.Factory {
		
		    @Override
		    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
		        return new CustomCallAdapter();
		    }
		}

3. addConverterFactory

	处理转换请求数据和响应数据，改变接口返回值中的泛型类型，默认是ResponseBody。

### 四、原理分析

1. 在调用接口后返回的是Call对象，之后的enqueue方法是OkHttp的，而之前就是Retrofit的。

2. Retrofit对象通过Builder进行配置，然后创建，可以配置url，转换器等等。

3. 调用Retrofit的create方法，里面使用的是Java动态代理，最终会返回一个接口代理对象。

4. 调用接口代理对象的方法，根据Java动态代理原理，会调用到InvocationHandler的invoke方法，在该方法中，会去解析所调用的接口方法的注解，最终生成OkHttp所需要的Call对象。

5. 调用Call对象的enqueue方法执行网络请求。

### 五、性能分析

1. 创建Retrofit对象：首次20ms左右，因为内部会创建OkHttpClient对象，底层会涉及到ssl相关初始化，首次创建会比较耗时。

2. create动态代理对象：0~1ms，基本不耗时。

3. 调用接口方法：首次5ms左右，因为首次会去解析注解，会比较慢，后面有缓存，就基本不耗时了。

### 六、总结

Retrofit库代码量很少，功能也比较简单，主要就是用来简化网络请求的封装，它的整体设计思想和架构设计都值得借鉴。