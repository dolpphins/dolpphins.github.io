---
title: Gradle高级用法
date: 2020-12-20 22:09:29
tags:
---

### 一、自定义Gradle插件

步骤如下：

1. 创建空目录，命名HelloPlugin。

2. 进入HelloPlugin目录下，新建build.gradle，写如下脚本代码：

		apply plugin: 'groovy'
		apply plugin: 'maven'
		
		dependencies {
		    implementation gradleApi()
		    implementation localGroovy()
		}
		
		repositories {
		    google()
		    jcenter()
		}
		
		//将插件打包上传到本地maven仓库
		uploadArchives {
		    repositories {
		        mavenDeployer {
		            pom.groupId = 'com.mtan'
		            pom.artifactId = 'HelloPlugin'
		            pom.version = '1.0.0'
		            repository(url: uri('../repos'))
		        }
		    }
		}

3. 新增目录src/main/groovy/com/mtan/plugin，新建HelloPlugin.groovy类，代码：

		package com.mtan.plugin;
		
		import org.gradle.api.Plugin
		import org.gradle.api.Project
		
		public class HelloPlugin implements Plugin<Project> {
		
		    public void apply(Project project) {
		        println 'HelloPlugin apply'
		    }
		}

4. 新建目录src/main/resources/META-INF/gradle-plugins，新增com.mtan.plugin.properties文件，代码：

		implementation-class=com.mtan.plugin.HelloPlugin

5. 项目导入idea中，执行uploadArchives任务，即可完成插件上传。

### 二、自定义Task

1. 在com.mtan.plugin包下新建task包，然后新建TestTask，注意后缀必须是groovy，不是java，代码如下：

		package com.mtan.plugin.task;
		
		import org.gradle.api.DefaultTask;
		import org.gradle.api.tasks.TaskAction;
		
		class TestTask extends DefaultTask {
		
		    TestTask() {
		        group = 'kk'
		    }
		
		    @TaskAction
		    def doExecute() {
		        println '哈哈哈'
		    }
		}

2. 在HelloPlugin中添加TestTask对象到项目tasks中：

		package com.mtan.plugin
		
		import com.mtan.plugin.task.TestTask;
		import org.gradle.api.Plugin
		import org.gradle.api.Project
		
		public class HelloPlugin implements Plugin<Project> {
		
		    public void apply(Project project) {
		        println 'HelloPlugin apply'
		
		        project.tasks.create("mao", TestTask)
		
		    }
		}

这样一个自定义的Task就实现好了，其它项目引入该Gradle插件后，就可以直接执行该TestTask。

### 三、自定义Transform

Transform是在javac编译之后执行的，如果我们自己定义了Transform，那么会比系统预定义的Transform先执行，我们可以通过Transform实现修改字节码等操作。

1. 先依赖android gradle插件，注意可能需要换个仓库，如下：

		dependencies {
		    implementation gradleApi()
		    implementation localGroovy()
		    implementation "com.android.tools.build:gradle:3.3.2"
		}
		
		repositories {
		    google()
		    jcenter()
		}

2. 在com.mtan.plugin包下新建包transform包，新增TestTransform.groovy，代码：
		
		package com.mtan.plugin.transform;
		
		import com.android.build.api.transform.QualifiedContent;
		import com.android.build.api.transform.Transform;
		import com.android.build.api.transform.TransformException;
		import com.android.build.api.transform.TransformInvocation;
		import com.android.build.gradle.internal.pipeline.TransformManager;
		
		import java.io.IOException;
		import java.util.Set;
		
		public class TestTransform extends Transform {
		
		    @Override
		    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
		        super.transform(transformInvocation);
		        println 'TestTransform transform'
		    }
		
		    @Override
		    String getName() {
		        return this.class.name
		    }
		
		    @Override
		    Set<QualifiedContent.ContentType> getInputTypes() {
		        return TransformManager.CONTENT_CLASS
		    }
		
		    @Override
		    Set<? super QualifiedContent.Scope> getScopes() {
		        return TransformManager.SCOPE_FULL_PROJECT
		    }
		
		    @Override
		    boolean isIncremental() {
		        return false;
		    }
		}

3. 在HelloPlugin中添加TestTransform对象到transform链中：

		package com.mtan.plugin
		
		import com.android.build.gradle.AppExtension
		import com.mtan.plugin.task.TestTask
		import com.mtan.plugin.transform.TestTransform;
		import org.gradle.api.Plugin
		import org.gradle.api.Project
		
		public class HelloPlugin implements Plugin<Project> {
		
		    public void apply(Project project) {
		        println 'HelloPlugin apply'
		
		        def android = project.extensions.getByType(AppExtension)
		
		        project.tasks.create("mao", TestTask)
		        android.registerTransform(new TestTransform())
		    }
		}

这样一个自定义Transform就实现好了，其它项目引入该Gradle插件后，执行编译，就可以看到该Transform被执行了，打印了日志。