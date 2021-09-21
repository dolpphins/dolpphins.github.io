---
title: Java动态代理原理
date: 2021-02-18 22:09:29
tags:
---

### 一、前言

Java动态代理可以实现动态生成代理类，而不需要手动写代理类，在使用上比较方便，而代理模式可以用于实现统计，拦截逻辑等统一管理控制的目的，比如在Retrofit中就用到了动态代理。

### 二、用法

1. 先定义一个接口：
	
		interface Animal {
		    void eat();
		}

2. 然后动态生成代理类对象：

		Animal proxy = (Animal) Proxy.newProxyInstance(Animal.class.getClassLoader(), new Class[]{Animal.class},
	        (proxy1, method, args1) -> {
	            System.out.println("invoke, " + method.getName());
	            // 调用真实对象的方法
	            return null;
        });

3. 最后调用方法：

		proxy.eat();

	调用该方法时，就会跑到InvocationHandler的invoke方法中，然后在该方法中去调用真实对象的方法，或者做一些统计逻辑，拦截逻辑等等。

### 三、原理分析

调用newProxyInstance方法返回的proxy对象，打印出它的全限定名为com.$Proxy0，这个类就是动态生成的，目前只存在内存中，可以把Class对象保存为class文件：
	
	byte[] data = ProxyGenerator.generateProxyClass("com.$Proxy0.class", proxy.getClass().getInterfaces());
	saveClass(data);

然后反编译查看源码：

	public final void eat() {
	    try {
	        this.h.invoke(this, m3, null);
	    } catch (Error | RuntimeException e) {
	        throw e;
	    } catch (Throwable e2) {
	        UndeclaredThrowableException undeclaredThrowableException = new UndeclaredThrowableException(e2);
	    }
	}

这里的h，就是InvocationHandler对象，当我们调用proxy的eat方法时，就会调用h的invoke方法。

那么Java内部是如何生成这个代理类的呢？其实内部最终也是调用了ProxyGenerator的generateProxyClass方法，该方法能够为给定的接口生成对应的代理类，生成后会得到byte[]数组，然后再转成Class对象，如果要进一步了解生成细节，可以查看ProxyGenerator类的源码。