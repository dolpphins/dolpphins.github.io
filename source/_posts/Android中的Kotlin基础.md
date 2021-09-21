---
title: Android中的Kotlin基础
date: 2020-07-11 18:59:09
tags:
---

### 变量、常量、数组

定义变量使用var，定义常量使用val，Kotlin会根据值自动推断类型，不需要自行指定，如下：

	var a = 1
	val b = 2

当然你要指定也可以：

	var a: Int = 1

定义可空变量和常量：

	var android: String? = null
	val name: String? = null

!!强制调用，空时抛出空指针异常，?.为空时返回空，Elvis运算符?:表示空时赋予某个值：

	name!!.toString()
	name?.toString()
	name ?: "haha"

创建数组：

	val arr = arrayOf(1, 2 3) // 都有初始值，无固定类型
	val arr1: IntArray = intArray(1, 2, 3) // 都有初始值的Int数组
	val arr2 = arrayOfNulls<Int>(5) // 无初始值的Int数组
	val arr3 = emptyArray<Int>()

### 条件和循环语句

条件语句可以做为一个值：

	val name: String = if (count < 10) {
		"haha"
	} else {
		"hehe"
	}

when语句：

	when {
		count == 5 -> "aaa"
		cout < 10 -> "bbb"		
		else -> "ccc"
	}

Kotlin中循环有for，while和do...while三种，对于for可以用in进行遍历，比如：

	for (i in 1...3) {
		......
	}

注意kotlin中的return如果是在lambda表达式中，默认是针对外层函数返回，要实现只对lambda表达式返回，需要用return@xxx。

### 函数

普通函数定义：

	fun add(a: Int, b: Int): Int {
		return a + b
	}

匿名函数定义：

	fun(a: Int, b: Int) = a + b

高阶函数

### 属性初始化

可以在定义时就初始化：

	val index: Int = 12

也可以在init块中初始化：

	val index: Int
	init {
		index = 12
	}

对于类似View引用的属性的初始化：

	private var mView: View? = null

或者：

	private lateinit var mView: View

最后在填充布局后初始化：

	mView = findViewById(R.id.xxx)

### 类


### 协程



### 其它

（1）使用字符串模板：

	val s = "kotlin"
	println("hello $s") // 打印hello kotlin

（2）使用is进行类型判断：

	val a = "hello kotlin"
	if (a is String) {
		println(a.length)
	}

注意if里面可以直接当成具体的类型对象使用，不需要再进行强制类型转换，判断不是某种类型用!is。


