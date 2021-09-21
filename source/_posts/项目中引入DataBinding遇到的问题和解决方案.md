---
title: 项目中引入DataBinding遇到的问题和解决方案
date: 2021-02-27 22:09:29
tags:
---

### 一、背景

在之前开发微信小程序时，发现数据绑定用起来比较顺手，代码也变得更简洁，因此抽空研究了Android中DataBinding数据绑定技术的使用和内部原理，该库算比较轻量级，也不需要依赖到很多其它组件，就决定在项目业务中尝试引入使用。

### 二、项目结构

项目结构分为宿主和插件两部分，所有插件的代码和资源都是独立的，而宿主是共用的，插件可以调用宿主的代码，反过来不行。在最终编译出来的apk中，宿主会打包到dex中，而每一个插件都打包成一个apk形式的包，然后通过DexClassLoader加载进来。

### 三、在插件中引入DataBinding

#### 引入过程

首先按照官方的文档，直接在插件的build.gradle中声明开启DataBinding：

	android {
	    dataBinding {
	        enabled true
	    }
	}

这里有个小坑，就是minSdkVersion必须>=14，这个很好解决，低的话改成14就可以了，然后本地编译，一切都是正常的，再尝试写布局，做数据绑定，引用相关类，都是没问题的，实际运行起来也是正常。

但这时提交代码，打发布包，就直接编译不过了，错误日志如下：

	...... conflict class ......
	2020-11-27 18:03:06:560 : android/support/v4/graphics/drawable/DrawableWrapper.class
	2020-11-27 18:03:06:560 : android/support/v4/app/TaskStackBuilder$SupportParentable.class
	......

意思就是类冲突了，项目很久之前就已经引入了v4包，而DataBinding的依赖库实际上也会依赖到v4包，这时就会有两份v4包代码，实际上DataBinding内部也有做相关处理来解决该问题，就是当发现项目已经implementation引入了v4包时，DataBinding会自动不再引入v4包，但项目引入v4包的并不是采用这种implementation的方式，它是自己把v4包的类打成几个jar包，然后放到lib目录下，这就导致了DataBinding还是会引入v4包，在最后打包的时候就出现了类冲突的问题，因为项目插件需要依赖v4包和改造成本大等问题，把v4包还原为implementation方法依赖并不可行。

这里一开始会想到初步的解决方法，原来的v4包最后是打到宿主dex中的，而目前DataBinding是在插件中使用，如果把DataBinding的v4包打到插件中，使用的时候也用插件中的，问题就解决了，但这个解决方法会存在一些问题，按照项目之前设定的类加载机制，会优先从宿主中找，没有才去插件中找，因此就必须修改这个查找规则，这个比较麻烦，另外一个问题就是如果以后宿主也要用DataBinding，那么这种方法就行不通了。

因此还是需要寻找方法排除掉DataBinding的v4包，这里的关键问题是因为DataBinding的依赖是自动添加的，我们无法干预，如果依赖是我们自己手动添加，那么加个exclude，把v4包排除掉，就解决了，因此就想能否不要再用那个脚本开关，改成自己手动依赖DataBinding的那几个库，如下：

	android {
	    defaultConfig {
	        minSdkVersion 14
	    }
	}
	
	dependencies {
	    annotationProcessor ("androidx.databinding:databinding-compiler:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	    implementation ("com.android.databinding:baseLibrary:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	    implementation ("com.android.databinding:library:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	    implementation ("com.android.databinding:adapters:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	
	}

编译试了一下，这种方法是不行的，原因比较简单，因为DataBinding有一个很关键的操作，就是会对布局做转换，把根节点为layout的布局转成正常的布局，而这个操作，实际上是由Android Gradle插件中的几个Task完成的，当判断到DataBinding开关为true时，才就会去执行这些task，而我们这里只是依赖了几个库，并不会触发执行这些task，在打开DataBinding开关后，观察build日志可以看到这些task：
	
	dataBindingExportBuildInfoDefaultDebug
	dataBindingExportBuildInfoDefaultRelease
	dataBindingExportFeaturePackageIdsDefaultDebug
	dataBindingExportFeaturePackageIdsDefaultRelease
	dataBindingGenBaseClassesDefaultDebug
	dataBindingGenBaseClassesDefaultRelease
	dataBindingMergeDependencyArtifactsDefaultDebug
	dataBindingMergeDependencyArtifactsDefaultRelease

很明显，就是这些task做了布局的转换，依赖的自动引入等工作，因为Android Gradle插件的逻辑修改不了，看了DataBindingOptions也没有对应的参数可以控制为不自动引入依赖包但要执行这些task，所以这里最终还是要把DataBinding开关打开，然后对自动添加的依赖做手脚，最终要能做到把DataBinding自动引入的依赖移除掉，或者为自动引入的依赖添加exclude排除掉v4包。

为了实现该目的，要先看一下Android Gradle插件是在哪里做自动引入DataBinding依赖的，先在dependencies里依赖gradle插件，这样才能愉快地查看gradle源码：

	dependencies {
	    ......
	    implementation "com.android.tools.build:gradle:3.3.2"
	}

然后在BasePlugin中，找到了如下关键代码：
	
	taskManager.addDataBindingDependenciesIfNecessary(
	                extension.getDataBinding(), variantManager.getVariantScopes());

addDataBindingDependenciesIfNecessary方法：
	
	public void addDataBindingDependenciesIfNecessary(
	            DataBindingOptions options, List<VariantScope> variantScopes) {
	        if (!options.isEnabled()) {
	            return;
	        }
	        boolean useAndroidX = globalScope.getProjectOptions().get(BooleanOption.USE_ANDROID_X);
	        String version = MoreObjects.firstNonNull(options.getVersion(),
	                dataBindingBuilder.getCompilerVersion());
	        String baseLibArtifact =
	                useAndroidX
	                        ? SdkConstants.ANDROIDX_DATA_BINDING_BASELIB_ARTIFACT
	                        : SdkConstants.DATA_BINDING_BASELIB_ARTIFACT;
	        project.getDependencies()
	                .add(
	                        "api",
	                        baseLibArtifact + ":" + dataBindingBuilder.getBaseLibraryVersion(version));
	        project.getDependencies()
	                .add(
	                        "annotationProcessor",
	                        SdkConstants.DATA_BINDING_ANNOTATION_PROCESSOR_ARTIFACT + ":" + version);
	        // TODO load config name from source sets
	        if (options.isEnabledForTests()
	                || this instanceof LibraryTaskManager
	                || this instanceof MultiTypeTaskManager) {
	            project.getDependencies()
	                    .add(
	                            "androidTestAnnotationProcessor",
	                            SdkConstants.DATA_BINDING_ANNOTATION_PROCESSOR_ARTIFACT
	                                    + ":"
	                                    + version);
	        }
	        if (options.getAddDefaultAdapters()) {
	            String libArtifact =
	                    useAndroidX
	                            ? SdkConstants.ANDROIDX_DATA_BINDING_LIB_ARTIFACT
	                            : SdkConstants.DATA_BINDING_LIB_ARTIFACT;
	            String adaptersArtifact =
	                    useAndroidX
	                            ? SdkConstants.ANDROIDX_DATA_BINDING_ADAPTER_LIB_ARTIFACT
	                            : SdkConstants.DATA_BINDING_ADAPTER_LIB_ARTIFACT;
	            project.getDependencies()
	                    .add("api", libArtifact + ":" + dataBindingBuilder.getLibraryVersion(version));
	            project.getDependencies()
	                    .add(
	                            "api",
	                            adaptersArtifact
	                                    + ":"
	                                    + dataBindingBuilder.getBaseAdaptersVersion(version));
	        }
	        project.getPluginManager()
	                .withPlugin(
	                        "org.jetbrains.kotlin.kapt",
	                        appliedPlugin -> {
	                            configureKotlinKaptTasksForDataBinding(project, variantScopes, version);
	                        });
	    }

上面这个方法，就是往dependencies中添加DataBinding依赖的逻辑，当然这个方法的代码不可修改，所以我们并不能直接把代码删掉就完事，但我们可以在这段代码执行后，去修改这个dependencies，在project.afterEvaluate中，找到api对应的ConfigurationContainer，然后获取到对应的dependencies，遍历它，把这几个自动引入的依赖都移除掉，然后我们自己在dependencies中声明依赖，如下：

	android {
	    defaultConfig {
	        minSdkVersion 14
	    }
	    dataBinding {
	        enabled true
	    }
	}
	
	project.afterEvaluate {
	    Set<?> set = new HashSet<>();
	    project.configurations["api"].getDependencies().each { item ->
	        if (item.group != null && item.group.toLowerCase().contains("databinding")) {
	            set.add(item)
	        }
	    }
	    project.configurations["api"].getDependencies().removeAll(set)
	}
	
	dependencies {
	
	    annotationProcessor ("androidx.databinding:databinding-compiler:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	
	    implementation ("com.android.databinding:baseLibrary:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	    implementation ("com.android.databinding:library:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	    implementation ("com.android.databinding:adapters:3.3.2") {
	        exclude group: 'com.android.support', module: 'support-v4'
	        exclude group: 'com.android.support', module: 'support-compat'
	        exclude group: 'com.android.support', module: 'support-core-ui'
	        exclude group: 'com.android.support', module: 'support-core-utils'
	        exclude group: 'android.arch.lifecycle', module: 'runtime'
	    }
	}

这样问题就解决了，关键就是移除掉了DataBinding自动引入的依赖，然后自己手动声明依赖，同时声明exclude掉v4包。

问题解决了，但上面的方法其实还可以继续优化，因为DataBinding自动引入的依赖的版本号都是跟Android Gradle插件绑定的，现在我们自己手动声明了依赖，写死了版本号，如果后面升级了Android Gradle插件，DataBinding的那几个Task逻辑改了，跟我们写死版本号的依赖库不兼容了，就会出问题，这时需要升级版本号比较麻烦，而且可以看到我们自己写的依赖代码也是比较多而且相似的，因此这里可以不要自己手动声明依赖了，而是改成为自动添加的依赖加上exclude，如下：

	android {
	
	    defaultConfig {
	        minSdkVersion 14
	    }
	
	    dataBinding {
	        enabled true
	    }
	}
	
	project.afterEvaluate {
	
	    // ============= 处理dataBinding自动添加的依赖 begin =============
	    Set<?> set = new HashSet<>();
	    Set<String> artifacts = new HashSet<>();
	    project.configurations["api"].getDependencies().each { item ->
	        if (item.group != null && item.group.toLowerCase().contains("databinding")) {
	            set.add(item)
	            artifacts.add(item.group + ":" + item.name + ":" + item.version)
	        }
	    }
	    project.configurations["api"].getDependencies().removeAll(set)
	    // 再把依赖添加回去，带上exclude
	    artifacts.each { artifact ->
	        project.getDependencies().add("api", artifact, {
	            exclude group: 'com.android.support', module: 'support-v4'
	            exclude group: 'com.android.support', module: 'support-compat'
	            exclude group: 'com.android.support', module: 'support-core-ui'
	            exclude group: 'com.android.support', module: 'support-core-utils'
	            exclude group: 'android.arch.lifecycle', module: 'runtime'
	        })
	    }
	    // ============= 处理dataBinding自动添加的依赖 end =============
	}

### 四、在宿主中引入DataBinding

上面所说的是在插件中引入使用，已经能够正常使用了，包大小增加40KB，但考虑到如果多个插件需要使用，甚至宿主也要用到，那么就会存在多份40KB重复代码，因此就考虑把它迁移到宿主中，给宿主和其它插件共同使用。

#### 引入过程

1. 因为宿主和插件都是独立的Gradle项目，因此不能直接把配置迁移到宿主的build.gradle中，因为这时生效的只是宿主，插件并不能用，插件的布局文件，注解也不会被自动处理转换。

2. 因此对于需要使用DataBinding的插件，还是需要在build.gradle中把DataBinding开关打开，按照之前所说，这里就会自动引入DataBinding的三个依赖库和一个注解处理器，因为注解处理器也是不能跨project的，而且只在编译时使用，因此不需要做特殊处理，而那三个依赖库，需要手动把它们移除掉，然后在宿主中引入依赖，插件代码中使用的都是宿主中引入的依赖库，这样所有插件就都是使用宿主中的依赖库了，也就不会出现每个插件都有一份DataBinding依赖库的问题，但实际编译代码，这里就会报错：

		****/ data binding error ****msg:Cannot find the setter for attribute 'android:onClick' with parameter type lambda on ......

	这个报错的原因，就是DataBinding的相关Task没有正常执行，导致布局文件没有转换成功，因此编译时报错了，说明这些Task必须依赖那三个依赖库，才能正常执行。

3. 因此就不再把插件的依赖库移除掉，而改成compileOnly依赖方式，再次编译，还是报错：

		Android dependency 'com.android.databinding:adapters:3.3.2' is set to compileOnly/provided which is not supported

	说明DataBinding的依赖库不支持compileOnly，compileOnly的方法行不通。

4. 虽然compileOnly不支持，但我们可以通过Transform来实现移除class的目的，自定义Transform，然后对于这个三个依赖库，都不再写入到输出目录下，这样插件就能编译成功，而且最后编译出来的插件也没有DataBinding相关类，但apk实际运行时报错：

		java.lang.NoClassDefFoundError: android.databinding.DataBinderMapperImpl ......

	这里需要看下DataBindingUtil的源码，它里面会创建一个DataBinderMapperImpl对象，而这个DataBinderMapperImpl类，是在编译时自动生成的，因此它在插件包中，是插件的ClassLoader加载的，但DataBindingUtil是在宿主中的，是宿主的ClassLoader加载的，插件的父ClassLoader是宿主的ClassLoader，所以这里就报NoClassDefFoundError。

5. 对于上面的问题，简单的方法就是把DataBindingUtil特殊处理，放到插件中，但实际运行又报其它类NoClassDefFoundError，因此该方法也不可行，即使最后能跑起来后期风险也会比较大。

6. 经过上面的尝试，由于DataBinding的特殊性，直接把三个依赖库放到宿主中不太可行，因此换个思路，还是在插件中引入依赖，然后想方法精简引入的代码量。

7. 由于历史原因，proguard文件中keep住了所有android开头的类，所以DataBinding的所有类都被原封不动地打到插件包里，但实际上DataBinding的类是可以proguard的，内部并没有用到反射等特性，这里并不能直接把之前的keep去掉，因为这样可能会导致其它地方被proguard掉后出问题，由于项目巨大历史悠久也不太可能重新去梳理所有android开头的类，这里最好最快的方法就是，在keep住所有android开头的类前提下，又指定databinding的类需要proguard，实现方法如下：

		-keep class !android.databinding.**, android.** {
		   *;
		}

**最终引入的代码量从40KB减少到9KB，基于改造成本等综合考虑，最后的解决方法就是在每个使用到DataBinding的插件中都引入这一份代码。**

### 五、总结

1. 对于这种通过开启脚本开关，然后会自动引入的依赖，要想做二次处理，都可以采用这种方法实现。

2. 解决上面的问题，要对gradle的依赖关系比较熟悉，可以通过阅读gradle源码和打印日志验证等方式，理清dependencies存储的数据结构。