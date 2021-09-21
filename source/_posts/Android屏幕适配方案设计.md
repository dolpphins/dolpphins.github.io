---
title: Android屏幕适配方案设计
date: 2019-11-24 22:11:14
tags:
---

## 相关概念

* 屏幕分辨率

	经常说的1080*1920，表示宽高的像素个数。

* 屏幕尺寸

	手机对角线的尺寸，单位是英寸，1英寸=2.54cm。

* dpi

	每英寸的像素个数，比如1080*1920的手机，5.5英寸，那么dpi为sqrt(1080*1080+1920*1920)/5.5=354
	
	而这里，Android定义了一系列的密度类型，比如mdpi为160，hdpi为240，xhdpi为320，xxhdpi为480。

* dp

	密度无关像素，1dp的像素数跟dpi有关，1dp=dpi/160。

* sp

	独立比例像素，跟dp类似，但它还跟字体大小相关联。


## 资源文件夹命名

* sw<N\>dp

	最小宽度限制，比如layout-sw600dp表示当屏幕最小宽度至少为600dp时才会使用该资源。这里的最小宽度，是指屏幕宽高较小边，也就是固定不变的。

* w<N\>dp

	最小宽度限制，跟sw类似，区别就是这里的宽度就是屏幕宽度，屏幕方向改变，宽度也会随之改变。

* h<N\>dp

	最小高度限制，比较少用。

* 具体分辨率

	如values-1920*1080，采用向下匹配的方式查找。

## 资源文件查找

#### drawable查找

一台hdpi的手机，drawable的查找顺序如下：

hdpi --> xhdpi --> xxhdpi --> xxxhdpi --> nodpi --> mdpi --> ldpi --> ldpi --> drawable

也就是优先从高但最接近的dpi中查找，nodpi结束；没有再从低但最接近的dpi查找，drawable结束。

另外，如果一个手机的dpi为354，那么它落在320~480的区间，对应为xhdp。

#### drawable内存占用

假如一张图片100*100，只放在xhdpi中，手机为xxhdpi，那么加载到内存中该图片会被放大，宽高放大的倍数为：手机dpi/xhdpi，放大可能图片也会变模糊。建议图片按主流手机的dpi切图，然后放到较高dpi的目录下，比如xxdpi目录。

#### values具体分辨率查找

经常会有values，values-1920x1080，values-1280x720这些文件夹，查找的规则就是往下查找，values放最后面，如果没有就停止，不会往上找，这个与drawable不一样。

## 适配方案

### 1. 适配技巧

* 不硬编码px，尽量使用dp，sp，wrap_content，match_parent。

* 使用.9图，矢量图，代码自定义View，动态计算。

* dimens适配多个sw文件夹
	
	       ├── res
	       ├── ├──values
	       ├── ├──values-sw320dp
		   ├── ├──values-sw360dp
		   ├── ├──values-sw400dp
		   ├── ├──values-sw420dp
		   ├── ├──values-sw460dp
		   ├── ├──...
	       ├── ├──values-sw560dp
		   ├── ├──values-sw600dp
		   ├── ├──values-sw640dp
	
	或者是特定分辨率文件夹

	       ├── res
	       ├── ├──values
	       ├── ├──values-480x320
		   ├── ├──values-800x480
		   ├── ├──values-960x540
		   ├── ├──values-1024x600
		   ├── ├──values-1280x720
		   ├── ├──...
	       ├── ├──values-1920x1080
		   ├── ├──values-2560x1440
	
	对于这两种方式的每一个values文件，定义一系列的值。具体可以参考：[https://github.com/ladingwu/dimens_sw](https://github.com/ladingwu/dimens_sw "https://github.com/ladingwu/dimens_sw")。

	这种适配方式的缺点就是维护起来麻烦，多文件增大apk大小。

## 今日头条适配方案

### 核心思想

动态修改DisplayMetrics的density值，修改的公式为：

density = 手机屏幕宽度(px) / 设计图宽度(dp)

这样用dp指定宽度就会自动按比例分配。


### 原理

Android中View的大小最终都会转为px，而转换的逻辑就在TypeValue的applyDimension方法中：

	public static float applyDimension(int unit, float value,
                                       DisplayMetrics metrics) {
        switch (unit) {
        case COMPLEX_UNIT_PX:
            return value;
        case COMPLEX_UNIT_DIP:
            return value * metrics.density;
        case COMPLEX_UNIT_SP:
            return value * metrics.scaledDensity;
        case COMPLEX_UNIT_PT:
            return value * metrics.xdpi * (1.0f/72);
        case COMPLEX_UNIT_IN:
            return value * metrics.xdpi;
        case COMPLEX_UNIT_MM:
            return value * metrics.xdpi * (1.0f/25.4f);
        }
        return 0;
    }

可以看到这里value * metrics.density，所以我们根据设计图修改density，这里计算的结果就能按照设计图的比例进行分配。

### 实现

因为DisplayMetric对象是在Resource对象中，因此对于每一个Context中的Resource对象，我们都需要修改其density值，另外，densityDpi和scaledDensity也需要修改，这几个值的含义如下：

* density

	1dp占用的像素个数，也就是dpi/160的值

* densityDpi

	屏幕的dpi，也就是1英寸占用的像素个数

* scaledDensity

	1sp占用的像素个数，正常情况下跟1dp一样，但会根据系统字体调整


### 优缺点

* 优点

	简单，适用范围广，没有用到反射

* 缺点

	可能会对旧布局产生影响，因为density被改了。

