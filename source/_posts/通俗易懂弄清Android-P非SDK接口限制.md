---
title: 通俗易懂弄清Android P非SDK接口限制
date: 2020-03-22 12:20:21
tags:
---

## Android P非SDK接口限制

### 1. 三种名单

* 黑名单：不可使用，否则会抛出异常。

* 灰名单：如果当前apk的API级别<接口的限制API级别，那么可以用，否则抛出异常。

* 白名单：受支持的接口，可以使用。

### 2. 抛出异常

如果使用了黑名单的接口，就会抛出异常，调用getDeclaredField()方法会抛出NoSuchFieldException，调用getDeclaredMethod()方法会抛出NoSuchMethodException。

如果是调用了getFields()方法或者getDeclaredFIelds()方法或者getMethods()方法或者getDeclaredMethods()方法，不会抛出异常，但返回的结果都不会包含非SDK接口对应的Field或者Method。

### 3. 日志

如果使用了黑名单接口，除了会抛出异常外，还会有一行warn级别的日志，日志格式为：

	Accessing hidden method Landroid/content/Intent;->addMiuiFlags......
	
### 4. 分析apk

可以去官网下载veridex工具，然后执行命令：

	./appcompat.sg --dex-file=apk路径
	
就可以自动分析apk中用到的哪些接口是在黑名单或者灰名单里，如下：

    ......
	20 hidden API(s) used: 0 linked against, 20 through reflection
	13 in greylist
	1 in blacklist
	1 in greylist-max-o
	5 in greylist-max-p
	
注意该分析针对的是apk包，也就是编译后的所有代码（包括依赖到的第三方库代码），而且第三方ROM厂商可能会加入新的黑名单接口，这些工具无法检测。

### 5. 限制原理

其实就是在Method或者Field元信息的标记位access_flags写上限制信息，然后在反射的时候做检查，另外如果调用所在的类的类加载器为BootClassLoader，也会放行。

## 绕开方法

1. 在C++层搜索内存，把对应的Method或者Field元信息的标记位修改为公开API接口对应的值。

2. 把调用非SDK接口对应的Class对象的ClassLoader设置为BootClassLoader，这样系统就不会做出限制了，不过这种直接修改ClassLoader的容易出问题。

3. 其实Java层的VMRuntime中有一个豁免接口setHiddenApiExemptions，直接反射调用把豁免前缀传进入就可以，简单省事，如下：

	    try {
            Class<?> clazz = Class.forName("dalvik.system.VMRuntime");
            Method getDeclaredMethod = Class.class.getDeclaredMethod("getDeclaredMethod", String.class, Class[].class);
            Object vmRuntime = clazz.getDeclaredMethod("getRuntime").invoke(null);
            Method method = (Method) getDeclaredMethod.invoke(clazz, "setHiddenApiExemptions", new Class[]{String[].class});
            method.invoke(vmRuntime, new Object[]{new String[]{"L"}});
        } catch (Throwable t) {
            t.printStackTrace();
        }

	这里其实有个问题，setHiddenApiExemptions本身就是在黑名单里面，我们无法通过直接反射获取到Method，因此这里先拿到getDeclaredMethod的Method（它的ClassLoader为BootClassLoader），然后再通过它去拿到setHiddenApiExemptions的Method。

	同理，其实对于任意一个黑名单接口，我们都可以这样去做，也就不需要用到setHiddenApiExemptions这个方法了，只是每个都做就会比较繁琐。

4. 不要用反射了，直接调用，可能需要依赖一个对应的库，该库的目的只是为了编译通过生成相应的代码，当然被调用的方法实际上必须是public的。

这些方法了解就可以，最好不要真的去绕开这个限制。
        
        


