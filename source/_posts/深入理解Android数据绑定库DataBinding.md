---
title: 深入理解Android数据绑定库DataBinding
date: 2020-10-07 22:09:29
tags:
---

### 一、前言

DataBinding库其实出来挺久了，最近工作涉及到跨平台开发，接触到前端的响应式编程，比如微信小程序开发，Vue框架，使用起来还是比较方便的，开发效率也比较高，因此深入分析了Android中的DataBinding库的使用和原理。

### 二、使用

#### 1. 简单入门

先从最简单的入手，为了让项目支持使用DataBinding，需要在build.gradle中打开开关：

	android {
	    ......
	    dataBinding {
	        enabled true
	    }
	}

然后写个简单的布局文件：

	<?xml version="1.0" encoding="utf-8"?>
	<layout
	    xmlns:android="http://schemas.android.com/apk/res/android">
	    <data>
	        <variable
	            name="text"
	            type="String" />
	    </data>
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:orientation="vertical">
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:layout_margin="10dp"
	            android:text="@{text}"/>
	    </LinearLayout>
	</layout>

最后在Activity的onCreate中设置text的值：

	public class BasicActivity extends Activity {
	
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	
	        final ActivityBasicBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_basic);
	        binding.setText("hello world");
	    }
	}

这样最简单的数据绑定demo就写好了，代码通过setText设置text的值，UI就会自动进行刷新。

#### 2. 表达式

可以看到，在布局文件中，使用数据绑定用的是@{}符号，而中间就是具体的变量，其实这里也可以支持各种表达式的，比如+-/*%，&& || 数组访问等等，官方给出支持表达式如下：

	算术运算符 + - / * %
	字符串连接运算符 +
	逻辑运算符 && ||
	二元运算符 & | ^
	一元运算符 + - ! ~
	移位运算符 >> >>> <<
	比较运算符 == > < >= <=（请注意，< 需要转义为 &lt;）
	instanceof
	分组运算符 ()
	字面量运算符 - 字符、字符串、数字、null
	类型转换
	方法调用
	字段访问
	数组访问 []
	三元运算符 ?:

注意表达式如果涉及到字符串的，外层需要使用’’，而内层的字符串使用””，另外所有的<都必须写成&lt;。

#### 3. Observable

在上面的代码中，绑定的变量是一个String对象text，比较简单，但其实还可以绑定到更复杂的实体类中，比如user.name，如果User只是一个普通的实体类，我们修改里面的name，实际上并不会自动更新UI，而需要调用setUser才能更新，下面以RGB实体类为例进行说明：

	Random random = new Random();
	data.setR(random.nextInt(256));
	data.setG(random.nextInt(256));
	data.setB(random.nextInt(256));
	// 这里必须重新设置才能触发更新
	mBinding.setData(data);

这时我们可以改造RGB类，让它继承自BaseObservable，然后在get方法上加注解@Bindable，在set方法中调用notifyPropertyChanged通知字段发生改变，如下：

	public class ObservableRGB extends BaseObservable {
	
	    private int r;
	    private int g;
	    private int b;
	
	    @Bindable
	    public int getR() {
	        return r;
	    }
	
	    public void setR(int r) {
	        this.r = r;
	        notifyPropertyChanged(BR.r);
	    }
	
	    @Bindable
	    public int getG() {
	        return g;
	    }
	
	    public void setG(int g) {
	        this.g = g;
	        notifyPropertyChanged(BR.g);
	    }
	
	    @Bindable
	    public int getB() {
	        return b;
	    }
	
	    public void setB(int b) {
	        this.b = b;
	        notifyPropertyChanged(BR.b);
	    }
	}

这时就不再需要调用setData重新设置整个实体类对象了，只要r或者g或者b的值改变了，都会字段更新到UI上。

这种做法还是比较麻烦的，因为rgb都是int值，其实有更简单的方法，也就是用ObservableInt，直接把rgb都定义为ObservableInt引用就可以实现了，如下：

	public class IntRGB {
	
	    private ObservableInt r = new ObservableInt();
	    private ObservableInt g = new ObservableInt();
	    private ObservableInt b = new ObservableInt();
	    private ObservableField<String> s = new ObservableField<>();
	
	    /**
	     * 注意这里必须返回ObservableInt对象，而不是int值
	     * @return
	     */
	    public ObservableInt getR() {
	        return r;
	    }
	
	    public void setR(int r) {
	        this.r.set(r);
	    }
	
	    public ObservableInt getG() {
	        return g;
	    }
	
	    public void setG(int g) {
	        this.g.set(g);
	    }
	
	    public ObservableInt getB() {
	        return b;
	    }
	
	    public void setB(int b) {
	        this.b.set(b);
	    }
	
	    public ObservableField<String> getS() {
	        return s;
	    }
	
	    public void setS(String s) {
	        this.s.set(s);
	    }
	}

注意这里有个坑，getXXX必须返回ObservableInt对象，而不能返回具体的值，不然会出现UI不刷新的问题。

#### 4. 绑定事件

DataBinding也支持绑定事件，比如绑定onClick的回调方法，onWindowFocusChanged的回调方法等等，而且绑定时支持lambda表达式，回调方法支持自定义参数：

	<Button
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="方法引用方式"
	        android:onClick="@{eventHandler::onClick}"/>
	
	<Button
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    android:text="lambda表达式方式"
	    android:onClick="@{() -> eventHandler.onLambdaClick(context)}"
	    android:layout_marginTop="10dp"/>

这里的context是DataBinding内部实现好的，可以直接用，实际上就是对应的Activity对象。

#### 5. BindingAdapter

在最开始的例子中，我们可能会想，DataBinding是如何知道android:text的值，要用setText方法去动态设置，这里其实就涉及到绑定查找顺序，DataBinding会先查找对应的set方法，比如text，就找setText，color就找setColor，如果找到就可以直接用了，如果找不到，就遍历BindingMethod注解，这个注解会声明xml属性对应哪个方法，如果还是没有，就会找被BindingAdapter注解的方法，该注解会指定该方法对应的属性，如果还是没有，就编译失败报错。

BindingMethod用法：

	@BindingMethods({
	    @BindingMethod(type = TextView.class, attribute = "app:bindText", method = "setText")
	})
	public class BindingAdapterHelper {
	    ......
	}

这里bindText是自定义属性，通过BindingMethod注解，把它映射到setText方法上，最终会通过setText方法设置该值，注意BindingMethods和BindingMethod是一起使用的，BindingMethod是BindingMethods的值。

BindingAdapter用法：

	@androidx.databinding.BindingAdapter({"url"})
	    public static void loadImage(ImageView view, String url) {
	        Log.i(TAG, "url:" + url);
	        Glide.with(view.getContext()).load(url).into(view);
	}

这里又有一个自定义属性url，通过BindingAdapter注解，DataBinding最终会调用loadImage方法，然后我们在该方法里实现自己的逻辑，比如加载网络图片。

#### 6. BindingConversion

这个就比较简单了，可以实现特定类型的自定义转换，二次处理，只要xml中的属性的参数和返回值与其返回值被BindingConversion注解的方法的参数和返回值一值，就会调用该方法，拿到返回值后再进行设置。

	<TextView
	    android:layout_width="100dp"
	    android:layout_height="100dp"
	    android:background="@{color}"/>
	
	@androidx.databinding.BindingConversion
	public static Drawable convertStringToDrawable(int color) {
	    return new ColorDrawable(color);
	}

#### 7. 双向数据绑定

上面所说的都是单向绑定，也就是从data到UI，但DataBinding也是支持从UI到data的，比如输入框内容发生改变了，data的值也会跟着改变，SeekBar的进度发生改变了，data的值也跟着改变，为了实现从UI到data的绑定，需要使用@={}符号：

	<EditText
	    android:layout_width="match_parent"
	    android:layout_height="wrap_content"
	    android:text="@={data.text}"
	    android:layout_marginTop="20dp"/>
	<TextView
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    android:layout_marginTop="10dp"
	    android:text="@{data.text}"/>

可以看到，EditText和data.text实现了双向绑定，data.text的值发生了改变，会显示到EditText上，在EditText上直接修改文本，也会导致data.text变成修改后的文本，下面的进度条也是类似：

	<SeekBar
	    android:id="@+id/search_bar"
	    android:layout_width="match_parent"
	    android:layout_height="wrap_content"
	    android:progress="@={data.progress}"
	    android:layout_marginTop="50dp"/>
	
	<TextView
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    android:text="@{String.valueOf(data.progress)}"
	    android:layout_marginTop="20dp"
	    android:textSize="20sp"/>

#### 7. include和ViewStub

DataBinding支持对include进行数据绑定，值会被传到include实际的布局中，通过也支持ViewStub。

### 三、原理

DataBinding的实现原理不算复杂，通过反编译生成的apk，或者调试内部代码，大概就会明白其中的实现方法，可以分为两个阶段，编译阶段和运行阶段。

#### 编译阶段

在编译时，DataBinding编译器会处理layout为根节点的xml文件，然后生成正常的xml布局文件，还是最开始的demo，生成的xml布局文件如下：

	<?xml version="1.0" encoding="utf-8"?>
	<LinearLayout
	              xmlns:android="http://schemas.android.com/apk/res/android"
	              android:orientation="vertical" 
	              android:tag="layout/activity_basic_0" 
	              android:layout_width="match_parent"
	              android:layout_height="match_parent">
	    <TextView 
	              android:tag="binding_1" 
	              android:layout_width="wrap_content"
	              android:layout_height="wrap_content" 
	              android:layout_margin="10dp"/>
	</LinearLayout>

可以看到，这个多了tag信息，这个是DataBinding自动加上的，待会在运行阶段会用到，会通过该tag找到该View对应的绑定的值。

编译时DataBinding还会生成几个类，最关键的就是XXXBindingImpl类，这个类的类名默认是根据xml命名按照一定规律生成的，当然也可以在data标签下指定为其它名字，这个XXXBindingImpl类就是实现data和UI绑定的关键，在运行阶段会用到。

#### 运行阶段

在运行时，我们会调用如下代码：

	final ActivityBasicBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_basic);
	binding.setText("hello world");

这里的ActivityBasicBinding对象，其实就是ActivityBasicBindingImpl类对象，它会做几件事：

1. 在对象初始化时，寻找所有的View引用，然后保存起来。
2. 在data发送改变时，会用Choreographer发送一个回调任务，该任务会在UI线程中执行。
3. 上面所说的回调任务，就是执行UI刷新操作，最终会调用到ActivityBasicBindingImpl类的executeBindings方法，该方法会调用属性的对应的方法进行UI刷新。
看下ActivityBasicBindingImpl类的executeBindings方法：

		@Override
		protected void executeBindings() {
		    long dirtyFlags = 0;
		    synchronized(this) {
		        dirtyFlags = mDirtyFlags;
		        mDirtyFlags = 0;
		    }
		    java.lang.String text = mText;
		
		    if ((dirtyFlags & 0x3L) != 0) {
		    }
		    // batch finished
		    if ((dirtyFlags & 0x3L) != 0) {
		        // api target 1
		
		        androidx.databinding.adapters.TextViewBindingAdapter.setText(this.mboundView1, text);
		    }
		}

这里直接调用mboundView1的setText也是可以的，但可能会导致死循环，所以实际上还是调用了TextViewBindingAdapter的setText方法，这个TextViewBindingAdapter是官方提供的，也就是TextView的BindingAdapter，在setText里做了新旧值相等判断，不相等才需要更新UI。

### 四、总结

经过上面的使用和分析，已经较全面地学习DataBinding的使用和内部原理，能够在项目中很轻松地写出DataBinding代码，接下来分析下DataBinding的优点和存在的问题。

#### 优点

1. 可以避免无用的刷新，实现高效局部刷新，比如ListView或者RecyclerView某个Item的某个控件的刷新。

2. 避免重复调用findViewById，DataBinding内部帮我们调用了，而且只调用一次。

3. 能够实现MVVM模式，数据改变直接触发UI更新，VM不会持有View的引用，也不会做更新UI的逻辑，该开发模式同时也能节省不少代码，提高开发效率。

#### 缺点

1. 因为都是全自动绑定，出问题不好定位，调试不容易。

2. 可能会导致占用内存大。

3. 编译速度变慢。

#### 适用场景

1. 常规数据绑定，逻辑解耦。

2. 实现局部刷新，提高性能。

3. 加上特殊逻辑，比如ImageView加载图片时特殊逻辑。