---
title: Jetpack深入浅出（一）
date: 2021-05-05 22:09:29
tags:
---

### 一、Jetpack是什么

2017年I/O大会推出Android Architecture Component，2018年I/O大会推出Jetpack，Support包并入Jetpack中。

来自官方的解释：Jetpack是一个由多个库组成的套件，可帮助开发者遵循最佳做法，减少样板代码并编写可在各种Android版本和设备中一致运行的代码，让开发者精力集中编写重要的代码。

简单总结下，Jetpack = 最佳实践库 + 工具库 + 兼容库

### 二、Jetpack有哪些组件

为了方便学习，可以简单分为几大类：

* Activity，Appcompat，Fragment

* Lifecycle，LiveData，ViewModel，DataBinding

* Compose，Navigation，Paging，Constraintlayout，Recyclerview

* DataStore，Room，Hilt，Startup，Work，Collection

* CameraX，Media2

* ……

### 三、从Lifecycle说起

Lifecycle是一个生命周期感知组件，可以将依赖组件的代码从生命周期方法移入组件本身中，也就是在组件内部就可以感知到外部的生命周期，使用方法如下：

	public class MainActivity extends AppCompatActivity {
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        getLifecycle().addObserver(new LifecycleEventObserver() {
	            @Override
	            public void onStateChanged(@NonNull LifecycleOwner source,
	                    @NonNull Event event) {
	                Log.i("MainActivity", event.name());
	            }
	        });
	    }
	}

这里getLifecycle()方法会返回一个Lifecycle对象，AppCompatActivity的继承体系：

![](/images/1.png)

虚线框中的ComponentActivitiy内部就实现了Lifecycle的具体逻辑，原理如下图：

![](/images/2.png)

getLifecycle()方法返回的实际上是一个LifecycleRegistry对象，在该对象里，有个mObserverMap，保存所有的生命周期观察者，还有个mState，表示当前所处的状态，而ComponentActivity在被创建后，会添加一个空的Fragment用于感知生命周期，当生命周期方法被回调时，就去通知mObserverMap里所有观察者。

总体来看，原理并不复杂，主要就是：空Fragment + 观察者模式

而对于各种状态，官网有一张图：

![](/images/3.png)

一般情况下，最右边的STARED和RESUMED就认为是活跃状态，其它都是非活跃状态。

如果想用Lifecycle，但又不想依赖ComponentActivity那一大串东西，或者实际项目中根本就不是Activity，那可以直接复用LifecycleRegistry类，自己实现getLifecycle方法，然后在合适的时机设置状态，如下：

	public class LifecycleActivity extends Activity implements LifecycleOwner {
	
	    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
	
	    @NonNull
	    @Override
	    public Lifecycle getLifecycle() {
	        return mLifecycleRegistry;
	    }
	
	    @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        mLifecycleRegistry.setCurrentState(State.CREATED);
	    }
	}

另外，使用空Fragment感知生命周期，这是一个很好的解耦方法，不用外部去手动调用什么onCreate，onResume之类的方法。

### 三、继续说LiveData

LiveData是一种可观察的数据存储器类，具有生命周期感知能力，确保仅更新处于活跃生命周期状态的应用组件观察者，基本的使用方法如下：

	final MutableLiveData data = new MutableLiveData();
	    data.observe(MainActivity.this, new Observer() {
	        @Override
	        public void onChanged(Object o) {
	            Log.i("MainActivity", o.toString());
	        }
	});

也可以扩展LiveData，监听活跃和非活跃状态：

	public class ExtendLiveData extends MutableLiveData<String> {
	
	    @Override
	    protected void onActive() {
	        super.onActive();
	        // 有活跃观察者
	        Log.i("ExtendLiveData", "onActive");
	    }
	
	    @Override
	    protected void onInactive() {
	        super.onInactive();
	        // 没有活跃观察者
	        Log.i("ExtendLiveData", "onInactive");
	    }
	}

还可以在onChanged回调前做数据转换：

	LiveData<String> data1 = new MutableLiveData<>();
	    LiveData<Object> data2 = Transformations.map(data1, new Function<String, Object>() {
	
	        @Override
	        public Object apply(String input) {
	            return input + "123";
	        }
	    });
	    data2.observe(this, new Observer<Object>() {
	        @Override
	        public void onChanged(Object o) {
	            Log.i("MainActivity", o.toString());
	        }
	});

对所有数据都加上后缀123。

对于LiveData，有几点需要注意：

* Observe第一个参数就是LifecycleOwner对象。
* 内部会注册观察器观察生命周期。
* 当状态>=STARTED时才回调onChanged方法。
* 状态重新回到>=STARTED时会回调延迟的事件。
* 默认支持粘性事件。

原理图如下：

![](/images/4.png)

分为两条线，postValue是修改数据，会先缓存到mData中，然后根据mActive判断是否需要回调onChanged方法。observe是注册观察者，内部会通过Lifecycle感知生命周期，当处于活跃状态时把mActive设置为true，处于非活跃状态时把mActive设置为false。

总结下，LiveData采用观察者模式，可以实现在不同页面，不同业务代码中去观察某个数据的变化，而且还引入了感知生命周期，可以避免crash和一些异常情况。

另外，美团开源了LiveEventBus库，它是基于LiveData的EventBus，[https://github.com/JeremyLiao/LiveEventBus](https://github.com/JeremyLiao/LiveEventBus)。

### 四、再说下ViewModel

ViewModel是一个更简单的组件，之前我们创建一个普通的ViewModel对象：

	final DataViewModel vm = new DataViewModel(); 

换成使用ViewModel库后，创建普通的ViewModel对象：

	final DataViewModel vm = new ViewModelProvider(this).get(DataViewModel.class);

最终都是会得到一个ViewModel对象，而使用第二种方式创建的话，会在应用配置发生改变时，ViewModel对象能保持不变，其原理如下：

![](/images/5.png)

1. 内存缓存ViewModelStore对象，该对象内缓存ViewModel数据。

2. 获取时优先拿缓存，没有就用newInstance方法创建对象，所以这里必须保证有无参构造方法。

3. 页面销毁时需要清空缓存，避免内存泄露。

ViewModel组件的优点：

* 配置发生改变时数据不会丢失。
* 内存中共享同一份数据，Fragment之间可以共享数据。

### 五、最后说DataBinding

DataBinding就是用来做数据绑定的，通过编译时生成辅助类，运行时自动更新UI等技术实现数据绑定，参考之前的文章：

[深入理解Android数据绑定库DataBinding](https://mtancode.com/2020/10/07/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Android%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A%E5%BA%93DataBinding/)

[项目中引入DataBinding遇到的问题和解决方案](https://mtancode.com/2021/02/27/%E9%A1%B9%E7%9B%AE%E4%B8%AD%E5%BC%95%E5%85%A5DataBinding%E9%81%87%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98%E5%92%8C%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/)

### 六、实现MVVM模式

先贴官方的图：

![](/images/6.png)

这里分为三层：UI层，ViewModel层和Model层，而其中的依赖关系是：View层 -> ViewModel层 -> Model层，当ViewModel层的数据发生改变时，会通过DataBinding自动反映到UI上，而不是手动去找到对应的View然后设置UI。

下面使用代码实现最简洁的MVVM模式，View层：

	<?xml version="1.0" encoding="utf-8"?>
	<layout
	    xmlns:android="http://schemas.android.com/apk/res/android">
	    <data>
	        <variable
	            name="data"
	            type="com.tencent.databindingapp.mvvm.DataViewModel" />
	    </data>
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        android:orientation="vertical">
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:layout_margin="10dp"
	            android:text="@{data.text1}"/>
	    </LinearLayout>
	</layout>
	
	final ActivityMvvmBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_mvvm);
	    binding.setLifecycleOwner(this);
	    final DataViewModel vm = new ViewModelProvider(this).get(DataViewModel.class);
	    binding.setData(vm);

ViewModel层：

	public class DataViewModel extends ViewModel {
	
	    private DataModel mDataModel = new DataModel();
	
	    private MutableLiveData<String> text1 = new MutableLiveData<String>();
	
	    public MutableLiveData<String> getText1() {
	        return text1;
	    }
	
	    public void onCreate() {
	        // 模拟数据改变
	        new Thread(){
	
	            @Override
	            public void run() {
	                for (int i = 0; i < 10; i++) {
	                    String value = mDataModel.getData();
	                    text1.postValue(value);
	                    try {
	                        Thread.sleep(2000);
	                    } catch (Throwable t) {
	                        t.printStackTrace();
	                    }
	                }
	            }
	
	        }.start();
	    }
	}

ViewModel层：

	public class DataModel {
	
	    public String getData() {
	        Random random = new Random();
	        return random.nextInt(1000000) + "";
	    }
	}

显然，MVVM模式的代码架构非常清晰，不同层次的职责很明确，耦合性更低，具有更好的可扩展性。