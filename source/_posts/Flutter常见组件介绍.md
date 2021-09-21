---
title: Flutter常见组件介绍
date: 2021-09-21 21:19:18
tags:
---

## 前言

Flutter的组件很多，接下来介绍开发过程中常见的一些组件，先上类图：

![](/images/flutter_1.jpg)

## 常见组件

### 基本组件

* **Text**

	文本组件。
	
* **Image**

	图片组件。
	
* **RaisedButton、FlatButton、OutlineButton、IconButton**

	按钮组件，也可以实现样式自定义。
	
* **Icon**

	图标组件。
	
* **TextField**

	输入框组件。

### 容器组件

* **Container**

	可以指定宽高，背景颜色，边框样式，padding和margin等。
	
* **Flex**	
	
	弹性布局，是Row和Column的父类，有个direction属性。
	
* **Row**

	横向排列，继承自Flex，仅仅只是指定方向为horizontal，mainAxisSize默认为max，也就是填满主轴方向。

* **Column**

	竖向排列，继承自Flex，仅仅只是指定方向为vertical，mainAxisSize默认为max，也就是填满主轴方向。
	
* **Flexible**
	
	fit属性固定为FlexFit.loose，也就是尽可能填满剩余空间但可不填满。
	
* **Expanded**

	该组件必须用在Row，Column或者Flex内，有个弹性系数属性flex，继承自Flexible，fit属性固定为FlexFit.tight，也就是强制填满剩余空间。

* **Stack**

	类似Android中的FrameLayout，配合Positioned使用，对于未定位子组件，大小由fit参数决定，StackFit.loose表示子组件自己决定，StackFit.expand表示尽可能大；对于定位子组件，大小由定位参数决定。

### 可滚动组件

* **SingleChildScrollView**

	滚动布局，滚动方向为水平或者垂直。
	
* **ListView**

	有四种方法创建ListView对象，直接初始化children最简单，无复用机制，通过builder进行构建，separated构建带分割线，custom进一步实现自定义，都会有复用机制。

* **GridView**

	GridView和ListView类似，实现方格布局。

* **CustomScrollView**

	粘合ListView和GridView，实现更复杂的UI组合列表。

### 约束和定位组件

* **Align**

	用来调整子组件在父组件中的位置，属性为alignment，根据子组件的宽高确定自身的宽高，属性为widthFactor和heightFactor，假设子组件宽高为w和h，则Align组件的宽高分别是widthFactor * w和heightFactor * h。
	
* **Center**

	继承自Align，alignment固定为center。

* **Padding**

	设置padding。
	
* **ConstrainedBox**

	约束子组件的最小宽高，最大宽高。
	
* **SizedBox**

	继承自ConstrainedBox，约束子组件的宽高。

### 自定义约束和定位组件

* **CustomSingleChildLayout**

	定义代理对单子组件实现自定义大小约束，位置偏移等。

* **CustomMultiChildLayout**

	定义代理多子组件实现自定义大小约束，位置偏移等，子组件通过id进行标识。
	
### 其它组件

* **Opacity**

	指定透明度。

* **GestureDetector**

	手势监控，包括点击，双击，长按等。
	
## 总结

可以发现一些组件间是有关联的，比如Center是Align的特殊实现，总结规律能更高效地学习，而对于完全陌生的新组件，也可以通过阅读源码快速了解组件的使用方法和实现原理。
