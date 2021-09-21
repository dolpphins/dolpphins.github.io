---
title: 深入理解Java反射
date: 2019-12-07 22:31:56
tags:
---

## 前言

本文主要介绍Java反射的内部原理，反射慢的原因，如何优化反射。

## 源码分析

平时开发中反射用得最多的应该是方法调用，下面分析反射调用方法的内部原理，先看getDeclaredMethod方法：

    @CallerSensitive
    public Method getDeclaredMethod(String name, Class<?>... parameterTypes)
        throws NoSuchMethodException, SecurityException {
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
        Method method = searchMethods(privateGetDeclaredMethods(false), name, parameterTypes);
        if (method == null) {
            throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
        }
        return method;
    }

这里分为三步：

1. 检查权限。

2. 获取所有Method，这里有一个SoftReference缓存，缓存所有的Fields和Methods信息。

3. 查找对应的Method后复制副本返回。

因此可以发现getDeclaredMethods和getDeclaredMethod其实性能差别不大，都是会先获取到所有Method信息。

再看一下getMethod方法：

    @CallerSensitive
    public Method getMethod(String name, Class<?>... parameterTypes)
        throws NoSuchMethodException, SecurityException {
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
        Method method = getMethod0(name, parameterTypes, true);
        if (method == null) {
            throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
        }
        return method;
    }

getMethod0方法：

	private Method getMethod0(String name, Class<?>[] parameterTypes, boolean includeStaticMethods) {
	    MethodArray interfaceCandidates = new MethodArray(2);
	    Method res =  privateGetMethodRecursive(name, parameterTypes, includeStaticMethods, interfaceCandidates);
	    if (res != null)
	        return res;
	
	    // Not found on class or superclass directly
	    interfaceCandidates.removeLessSpecifics();
	    return interfaceCandidates.getFirst(); // may be null
    }

privateGetMethodRecursive方法：

    private Method privateGetMethodRecursive(String name,
            Class<?>[] parameterTypes,
            boolean includeStaticMethods,
            MethodArray allInterfaceCandidates) {
        // Note: the intent is that the search algorithm this routine
        // uses be equivalent to the ordering imposed by
        // privateGetPublicMethods(). It fetches only the declared
        // public methods for each class, however, to reduce the
        // number of Method objects which have to be created for the
        // common case where the method being requested is declared in
        // the class which is being queried.
        //
        // Due to default methods, unless a method is found on a superclass,
        // methods declared in any superinterface needs to be considered.
        // Collect all candidates declared in superinterfaces in {@code
        // allInterfaceCandidates} and select the most specific if no match on
        // a superclass is found.

        // Must _not_ return root methods
        Method res;
        // Search declared public methods
        if ((res = searchMethods(privateGetDeclaredMethods(true),
                                 name,
                                 parameterTypes)) != null) {
            if (includeStaticMethods || !Modifier.isStatic(res.getModifiers()))
                return res;
        }
        // Search superclass's methods
        if (!isInterface()) {
            Class<? super T> c = getSuperclass();
            if (c != null) {
                if ((res = c.getMethod0(name, parameterTypes, true)) != null) {
                    return res;
                }
            }
        }
        // Search superinterfaces' methods
        Class<?>[] interfaces = getInterfaces();
        for (Class<?> c : interfaces)
            if ((res = c.getMethod0(name, parameterTypes, false)) != null)
                allInterfaceCandidates.add(res);
        // Not found
        return null;
    }

流程还是一样，分为三步：

1. 检查权限

2. 获取所有public的Method（自身 --> 父类 --> 接口）

3. 查找对应的Method后复制副本返回

与getDeclaredMethod的区别，就是这里只获取public的Method，但却可以获取到父类，接口的Method。

**注：getDeclaredField和getField流程类似，只是把Method替换成Field，不再分析。**

继续分析Method的invoke方法：

    @CallerSensitive
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, obj, modifiers);
            }
        }
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    }

这里有个MethodAccessor对象，invoke最后是委托给它调用了，看一下acquireMethodAccessor方法：

   	private MethodAccessor acquireMethodAccessor() {
        // First check to see if one has been created yet, and take it
        // if so
        MethodAccessor tmp = null;
        if (root != null) tmp = root.getMethodAccessor();
        if (tmp != null) {
            methodAccessor = tmp;
        } else {
            // Otherwise fabricate one and propagate it up to the root
            tmp = reflectionFactory.newMethodAccessor(this);
            setMethodAccessor(tmp);
        }

        return tmp;
    }

同理，这里也是有缓存机制，MethodAccessor对象只有首次才会被创建，然后缓存起来，后面直接使用。看首次MethodAccessor是如何被创建的：

    public MethodAccessor newMethodAccessor(Method method) {
        checkInitted();

        if (noInflation && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            return new MethodAccessorGenerator().
                generateMethod(method.getDeclaringClass(),
                               method.getName(),
                               method.getParameterTypes(),
                               method.getReturnType(),
                               method.getExceptionTypes(),
                               method.getModifiers());
        } else {
            NativeMethodAccessorImpl acc =
                new NativeMethodAccessorImpl(method);
            DelegatingMethodAccessorImpl res =
                new DelegatingMethodAccessorImpl(acc);
            acc.setParent(res);
            return res;
        }
    }

大部分情况都是走else的逻辑，也就是说这个MethodAccessor，其实就是NativeMethodAccessorImpl对象，那我们直接看NativeMethodAccessorImpl的invoke方法：

    public Object invoke(Object obj, Object[] args)
        throws IllegalArgumentException, InvocationTargetException
    {
        // We can't inflate methods belonging to vm-anonymous classes because
        // that kind of class can't be referred to by name, hence can't be
        // found from the generated bytecode.
        if (++numInvocations > ReflectionFactory.inflationThreshold()
                && !ReflectUtil.isVMAnonymousClass(method.getDeclaringClass())) {
            MethodAccessorImpl acc = (MethodAccessorImpl)
                new MethodAccessorGenerator().
                    generateMethod(method.getDeclaringClass(),
                                   method.getName(),
                                   method.getParameterTypes(),
                                   method.getReturnType(),
                                   method.getExceptionTypes(),
                                   method.getModifiers());
            parent.setDelegate(acc);
        }

        return invoke0(method, obj, args);
    }

这里有个次数判断，ReflectionFactory.inflationThreshold()返回15，也就是说这里有Java和Native两个版本实现了invoke，前15次是调用了Native版本，第16次开始调用Java版本，这样做的原因是Native版本初始化快，但多次运行性能比Java版本的差，而Java版本刚好相反，初始化慢，但多次运行后性能比较好，因此就有了这个优化。

说一下这个Java版本的实现，它是动态生成了一个类，然后可以做一些调用优化了，包括方法内联，因此性能就会好很多，但生成类这个过程会导致初始化慢。

到这里基本流程就差不多了，至于Java版本和Native版本如何invoke，暂时不再深究。

## Java反射为什么慢

1. 访问速度：需要在运行时遍历找出对应的Field，Method等，然后进行控制检查，类型检查等，这些都需要消耗时间，而静态代码这些都在编译时就处理好了。

2. 参数类型：由于传入参数数组为Object[]，在实际调用会转成具体类型，或者拆箱装箱等。

3. JIT优化：大部分的JIT优化都做不了。

## Java反射优化方法

1. 缓存相关的对象，比如Class，Method，Field等对象。

2. setAccessible设置为true（关闭安全检查）也可以提升性能。

3. ReflectASM库的优化方法，先生成辅助类（只在内存中，耗时），然后直接调用方法。生成的类比如像这个：

		public class CatMethodAccess extends MethodAccess {
			public Object invoke(Object obj, int i, Object... objArr) {
				Cat cat = (Cat) obj;
				switch (i) {
					case 0:
						Cat.eat();
						return null;
					default:
						throw new IllegalArgumentException("Method not found: " + i);
				}
	    	}
		}

	其实就是把反射转为直接调用了，生成的类跟反射的类会在同个包下（如果是java开头的还会在前面加上reflectasm），所以有个限制，就是只能反射public或者包级别或者protected方法，不能反射private方法，可以认为能够直接调用的方法才能够反射。

## 参考资料

[关于反射调用方法的一个log](https://www.iteye.com/blog/rednaxelafx-548536 "https://www.iteye.com/blog/rednaxelafx-548536")

[深入浅出 JIT 编译器](https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/index.html "https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/index.html")

[Java JIT 知识](https://www.jianshu.com/p/eea12f3bf490 "https://www.jianshu.com/p/eea12f3bf490")