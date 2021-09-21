---
title: ARouter框架原理剖析之Activity跳转实现原理
date: 2020-06-06 22:40:11
tags:
---

### 一、简单使用

1. Application中初始化：

	    @Override
    	protected void attachBaseContext(Context base) {
        	super.attachBaseContext(base);
			......
        	ARouter.openLog();
        	ARouter.openDebug();
        	ARouter.init(this);
			......
    	}

2. 在目标Activity加上注解：

		@Route(path = "/activity/second")
		public class SecondActivity extends Activity {
		
			......
		}

3. 跳转到目标Activity：

		ARouter.getInstance().build("/activity/second").navigation();

	可以看到，对于实现Activity跳转，使用起来很简单，思路清晰，代码不多。

### 二、从navigation方法说起

navigation方法内部经过一系列的调用，最终会走到_ARouter的navigation方法，这里会传入一个Postcard对象，它代表一个跳转请求，携带了相关的请求参数和数据，不过这时还只是个空壳对象，这里会调用LogisticsCenter.completion方法进行数据填充，该方法代码如下：

	public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }

        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
            if (null == groupMeta) {
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                // Load route and cache it into memory, then delete from metas.
                try {
                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }

                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);
                    Warehouse.groupsIndex.remove(postcard.getGroup());

                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }
                } catch (Exception e) {
                    throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
                }

                completion(postcard);   // Reload
            }
        } else {
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            Uri rawUri = postcard.getUri();
            if (null != rawUri) {   // Try to set params into bundle.
                Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
                Map<String, Integer> paramsType = routeMeta.getParamsType();

                if (MapUtils.isNotEmpty(paramsType)) {
                    // Set value by its type, just for params which annotation by @Param
                    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                        setValue(postcard,
                                params.getValue(),
                                params.getKey(),
                                resultMap.get(params.getKey()));
                    }

                    // Save params name which need auto inject.
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }

                // Save raw uri
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }

            switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must implement IProvider
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }
        }
    }

Warehouse.routes保存加载后的路由元数据对象，Warehouse.groupsIndex保存路由元数据加载信息，进程启动后首次调用到这里，routes里面的缓存还是空的，所以这时就会从groupsIndex中拿到对应group的路由元数据加载信息，也就是一个Class对象，每一种group就对应一个Class类，这个类是编译时自动生成的，比如我现在要跳转Activity，对应的类为ARouter$$Group$$activity，拿到Class对象后，调用newInstance方法就可以创建其实例对象了，然后调用loadInto方法，看下ARouter$$Group$$activity的loadInto方法：

	@Override
  	public void loadInto(Map<String, RouteMeta> atlas) {
    	atlas.put("/activity/second", RouteMeta.build(RouteType.ACTIVITY, SecondActivity.class, "/activity/second", "activity", null, -1, -2147483648));
  	}

这个传进来的atlas就是Warehouse.routes对象，可以看到这里保存了路径和对应Activity的映射关系，当然这个代码是编译时自动生成的，我们只需要写个Route注解就可以。

再回到LogisticsCenter.completion方法，ARouter$$Group$$activity对象创建了，也调用它的loadInto方法了，这时再调用completion方法，这次拿到的RouteMeta对象就不为空了，因此走else逻辑，可以看到接下来会有一系列的setXXX操作，也就是把RouteMeta的数据填充到Postcard对象中，包括要跳转的目标Activity的Class等等。

Postcard对象填充完成后，就比较简单了，调用_navigation进行真正的跳转：

	private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                // Navigation in main looper.
                if (Looper.getMainLooper().getThread() != Thread.currentThread()) {
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            startActivity(requestCode, currentContext, intent, postcard, callback);
                        }
                    });
                } else {
                    startActivity(requestCode, currentContext, intent, postcard, callback);
                }

                break;
		......
    }

这里会创建Intent对象，然后把Postcard中的数据设置到Intent对象中，这些数据就是外部调用navigation方法处自行指定的，包括启动Activity的flags，传递的参数等等。

navigation方法的流程分析结束了，上面提到了Warehouse.groupsIndex保存路由元数据加载信息，那这个信息是什么时候添加到Warehouse.groupsIndex中的呢？还记得在Application中我们调用了init方法进行初始化，所以应该是这里面实现了添加操作，跟踪ARouter的init方法，最终走到LogisticsCenter的init方法：

	/**
     * LogisticsCenter init, load all metas in memory. Demand initialization
     */
    public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
        mContext = context;
        executor = tpe;

        try {
            long startInit = System.currentTimeMillis();
            //billy.qi modified at 2017-12-06
            //load by plugin first
            loadRouterMap();
            if (registerByPlugin) {
                logger.info(TAG, "Load router map by arouter-auto-register plugin.");
            } else {
                Set<String> routerMap;

                // It will rebuild router map every times when debuggable.
                if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                    logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                    // These class was generated by arouter-compiler.
                    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                    if (!routerMap.isEmpty()) {
                        context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                    }

                    PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
                } else {
                    logger.info(TAG, "Load router map from cache.");
                    routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
                }

                logger.info(TAG, "Find router map finished, map size = " + routerMap.size() + ", cost " + (System.currentTimeMillis() - startInit) + " ms.");
                startInit = System.currentTimeMillis();

                for (String className : routerMap) {
                    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        // This one of root elements, load root.
                        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                        // Load interceptorMeta
                        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                        // Load providerIndex
                        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                    }
                }
            }

            logger.info(TAG, "Load root element finished, cost " + (System.currentTimeMillis() - startInit) + " ms.");

            if (Warehouse.groupsIndex.size() == 0) {
                logger.error(TAG, "No mapping files were found, check your configuration please!");
            }

            if (ARouter.debuggable()) {
                logger.debug(TAG, String.format(Locale.getDefault(), "LogisticsCenter has already been loaded, GroupIndex[%d], InterceptorIndex[%d], ProviderIndex[%d]", Warehouse.groupsIndex.size(), Warehouse.interceptorsIndex.size(), Warehouse.providersIndex.size()));
            }
        } catch (Exception e) {
            throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
        }
    }

这里最关键的代码就是routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);这一行，它会把
com.alibaba.android.arouter.routes包下的所有类的全限定名找出来，然后添加到Warehouse.groupsIndex中，这里注意到有个startInit变量，看日志它是在打印这段逻辑的执行时间，说明这逻辑可能是比较耗时了，而我们是在attachBaseContext方法里进行调用的，初始化过慢就会导致程序启动慢，必要时需要进行优化（比如可以采用插桩进行优化）。

还有最后一个问题，就是ARouter是如何自动生成com.alibaba.android.arouter.routes包下的类，这个就要看arouter-compiler的源码了，找到RouteProcessor类，可以看到它用来处理Route和Autowired两个注解，所以它的流程是这样的：在编译时，根据javac的编译流程，会调用到RouteProcessor的process方法，这个方法就会获取到所有被加入Route注解的类，然后生成ARouter$$Group$$activity等对应的类，所在的包为com.alibaba.android.arouter.routes，这些自动生成的类，都实现了IRouteGroup接口，因此也就实现了loadInto方法，而这个方法中，比如ARouter$$Group$$activity，就会把路径和具体的Activity类的映射关系存到atlas中。

再细想一下，其实这个映射关系我们自己手写代码也是可以的，但就是写起来比较麻烦，也可能会写漏，每次新增一个Route注解的类，就要来这里写一下映射关系，时间久了就很容易忘记。

### 三、自己动手实现

弄清楚原理后，就可以自己动手实现一个了，流程如下：

1. 新增MRouterApp工程，做为测试项目。

2. 新建Android Library，命名为annotation，这个Module用来放注解，因为待会mrouter和compiler都会用到相关的注解，所以都会依赖到它，这里我们定义注解为Route：

		package com.mtan.mrouter.annotation;

		import java.lang.annotation.ElementType;
		import java.lang.annotation.Retention;
		import java.lang.annotation.RetentionPolicy;
		import java.lang.annotation.Target;
		
		@Retention(RetentionPolicy.CLASS)
		@Target(ElementType.TYPE)
		public @interface Route {
		
		    String path();
		}

3. 新建Java Library，命名为compiler，用于在编译时自动生成代码，首先在build.gradle引入相关第三方依赖库：

   		annotationProcessor 'com.google.auto.service:auto-service:1.0-rc3'
    	implementation 'com.google.auto.service:auto-service:1.0-rc3'
    	implementation 'com.squareup:javapoet:1.8.0'

	然后新建RouterProcessor类，继承自AbstractProcessor，实现相关方法：

		package com.mtan.mrouter.compiler;

		import com.google.auto.service.AutoService;
		import com.mtan.mrouter.annotation.Route;
		import com.squareup.javapoet.ClassName;
		import com.squareup.javapoet.JavaFile;
		import com.squareup.javapoet.MethodSpec;
		import com.squareup.javapoet.ParameterSpec;
		import com.squareup.javapoet.ParameterizedTypeName;
		import com.squareup.javapoet.TypeSpec;
		
		import java.util.HashSet;
		import java.util.Map;
		import java.util.Set;
		
		import javax.annotation.processing.AbstractProcessor;
		import javax.annotation.processing.Filer;
		import javax.annotation.processing.Messager;
		import javax.annotation.processing.ProcessingEnvironment;
		import javax.annotation.processing.Processor;
		import javax.annotation.processing.RoundEnvironment;
		import javax.lang.model.element.Element;
		import javax.lang.model.element.Modifier;
		import javax.lang.model.element.TypeElement;
		import javax.tools.Diagnostic;
		
		@AutoService(Processor.class)
		public class RouterProcessor extends AbstractProcessor {

		    private Messager messager;
		    private Filer mFiler;
		
		    @Override
		    public synchronized void init(ProcessingEnvironment processingEnvironment) {
		        super.init(processingEnvironment);
		        messager = processingEnv.getMessager();
		        mFiler = processingEnv.getFiler();
		    }
		
		    @Override
		    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
		        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(Route.class);
		        if (elements == null || elements.size() <= 0) {
		            return false;
		        }
		
		        ParameterizedTypeName inputMapTypeOfGroup = ParameterizedTypeName.get(
		                ClassName.get(Map.class),
		                ClassName.get(String.class),
		                ClassName.get(Class.class)
		        );
		
		        ParameterSpec param = ParameterSpec.builder(inputMapTypeOfGroup, "atlas").build();
		
		        MethodSpec.Builder loadIntoMethodOfRootBuilder = MethodSpec.methodBuilder("loadInto")
		                .addModifiers(Modifier.PUBLIC)
		                .addParameter(param);
		
		        for (Element e : elements) {
		            messager.printMessage(Diagnostic.Kind.NOTE, e.getSimpleName().toString());
		
		            String path = e.getAnnotation(Route.class).path();
		            ClassName className = ClassName.get((TypeElement) e);
		            loadIntoMethodOfRootBuilder.addStatement("atlas.put($S, $T.class)", path, className);
		        }
		
		
		
		        try {
		            String rootFileName = "MRouter$$Group$$activity";
		            JavaFile.builder("com.mtan.mrouter.apt",
		                    TypeSpec.classBuilder(rootFileName)
		                            .addJavadoc("NOT MODIFY!!!")
		                            .addModifiers(Modifier.PUBLIC)
		                            .addMethod(loadIntoMethodOfRootBuilder.build())
		                            .build()
		            ).build().writeTo(mFiler);
		        } catch (Throwable t) {
		            messager.printMessage(Diagnostic.Kind.ERROR, t.getMessage());
		        }
		
		        return true;
		    }
		
		    @Override
		    public Set<String> getSupportedAnnotationTypes() {
		        Set<String> annotations = new HashSet<>();
		        annotations.add(Route.class.getCanonicalName());
		        return annotations;
		    }
		}

	这里会自动生成一个类为MRouter$$Group$$activity，放在包com.mtan.mrouter.apt下，比如待会在测试项目MRouterApp中为SecondActivity类加了Route注解，那么最终生成的MRouter$$Group$$activity类的代码如下：

		package com.mtan.mrouter.apt;

		import com.mtan.mrouter.SecondActivity;
		import java.lang.Class;
		import java.lang.String;
		import java.util.Map;
		
		/**
		 * NOT MODIFY!!! */
		public class MRouter$$Group$$activity {
		  public void loadInto(Map<String, Class> atlas) {
		    atlas.put("/activity/second", SecondActivity.class);
		  }
		}

4. 新建Module，命名为mrouter，新建MRouter类：

		package com.mtan.mrouter;
		
		import android.app.Activity;
		import android.content.Context;
		import android.content.Intent;
		
		import java.lang.reflect.Method;
		import java.util.HashMap;
		import java.util.Map;
		import java.util.Set;
		
		public class MRouter {
		
		    private static Map<String, Class<?>> sRouteMap = new HashMap<>();
		
		    public static void init(Context context) {
		        try {
		            Set<String> names = ClassUtils.getFileNameByPackageName(context, "com.mtan.mrouter.apt");
		            for (String name : names) {
		                Class<?> clazz = Class.forName(name);
		                Object obj = clazz.newInstance();
		                Method method = clazz.getDeclaredMethod("loadInto", Map.class);
		                method.invoke(obj, sRouteMap);
		            }
		        } catch (Throwable t) {
		            t.printStackTrace();
		        }
		    }
		
		    private Context mContext;
		
		    public MRouter(Context context) {
		        mContext = context;
		    }
		
		    public void navigation(String path) {
		        Class<?> clazz = sRouteMap.get(path);
		        if (clazz == null) {
		            throw new RuntimeException("can't find class for path:" + path);
		        }
		        Intent intent = new Intent(mContext, clazz);
		        if (mContext instanceof Activity) {
		            mContext.startActivity(intent);
		        } else {
		            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
		            mContext.startActivity(intent);
		        }
		    }
		}

	ClassUtils类是从ARouter中复制过来的，用来加载com.mtan.mrouter.apt包下自动生成的类的名称。

5. 来到这里，MRouter库的代码就写好了，最后再梳理下这几个Module的依赖关系，annotation为公共库，不依赖其它Module，compiler会使用到相关注解，因此需要implementation依赖annotation，mrouter虽然目前不会使用到相关注解，但可能以后会使用到，而且使用mrouter的第三方工程也需要用到相关注解，因此mrouter也是需要依赖annotation，而且是api依赖，表示传递依赖。

6. 最后在MRouterApp中写测试代码，先把上面写好的库引入进来：

		implementation project(':mrouter')
    	annotationProcessor project(':compiler')

	然后新建App类，AndroidManifest指定Application为App类，在attachBaseContext中初始化：
	
		@Override
    	protected void attachBaseContext(Context base) {
        	super.attachBaseContext(base);
        	MRouter.init(this);
    	}

	接着新建SecondActivity类，加上Route注解：

		package com.mtan.mrouterapp;
		
		import android.app.Activity;
		
		import com.mtan.mrouterapp.annotation.Route;
		
		@Route(path = "/activity/second")
		public class SecondActivity extends Activity {
		    
		}

	最后在MainActivity中触发跳转到SecondActivity：

		findViewById(R.id.btn1).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                new MRouter(MainActivity.this).navigation("/activity/second");
            }
        });

7. 真机运行，测试能正常跳转，项目链接：[https://github.com/dolpphins/MRouter](https://github.com/dolpphins/MRouter "https://github.com/dolpphins/MRouter")。