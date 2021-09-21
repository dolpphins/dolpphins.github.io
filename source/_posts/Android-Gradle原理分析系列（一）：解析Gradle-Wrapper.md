---
title: Android Gradle原理分析系列（一）：解析Gradle Wrapper
date: 2019-12-21 21:26:49
tags:
---

## 前言

新建一个AS工程，编译器会自动帮我们创建了Gradle Wrapper相关的文件，目录结构如下：

	├── gradle
	│   └── wrapper
	│       ├── gradle-wrapper.jar
	│       └── gradle-wrapper.properties
	├── gradlew
	└── gradlew.bat

这里有几个问题：

1. 这个gradlew和gradle有什么区别？

2. 我们在gradle-wrapper.properties中配置的信息是如何被解析的？能配置的项又有哪些？

3. 如果网络不好，如何自己手动下载gradle压缩包配置版本库？

下面会分析Gradle Wrapper的实现原理，解答这几个问题。

## 从gradlew谈起

一般我们编译Android项目，不会直接执行gradle命令，而是执行gradlew xxxx，而这个gradlew，其实就是执行当前目录下的gradlew.bat（windows平台下），所以先看一下这个批处理文件究竟执行了什么命令：

	@rem Execute Gradle
	"%JAVA_EXE%" %DEFAULT_JVM_OPTS% %JAVA_OPTS% %GRADLE_OPTS% "-Dorg.gradle.appname=%APP_BASE_NAME%" -classpath "%CLASSPATH%" org.gradle.wrapper.GradleWrapperMain %CMD_LINE_ARGS%

这里会去运行/gradle/wrapper/目录下的gradle-wrapper.jar，然后执行其org.gradle.wrapper.GradleWrapperMain类的main方法：

	public static void main(String[] args) throws Exception {
        File wrapperJar = wrapperJar();
        File propertiesFile = wrapperProperties(wrapperJar);
        File rootDir = rootDir(wrapperJar);

        CommandLineParser parser = new CommandLineParser();
        parser.allowUnknownOptions();
        parser.option(GRADLE_USER_HOME_OPTION, GRADLE_USER_HOME_DETAILED_OPTION).hasArgument();
        parser.option(GRADLE_QUIET_OPTION, GRADLE_QUIET_DETAILED_OPTION);

        SystemPropertiesCommandLineConverter converter = new SystemPropertiesCommandLineConverter();
        converter.configure(parser);

        ParsedCommandLine options = parser.parse(args);

        Properties systemProperties = System.getProperties();
        systemProperties.putAll(converter.convert(options, new HashMap<String, String>()));

        File gradleUserHome = gradleUserHome(options);

        addSystemProperties(gradleUserHome, rootDir);

        Logger logger = logger(options);

        WrapperExecutor wrapperExecutor = WrapperExecutor.forWrapperPropertiesFile(propertiesFile);
        wrapperExecutor.execute(
                args,
                new Install(logger, new Download(logger, "gradlew", UNKNOWN_VERSION), new PathAssembler(gradleUserHome)),
                new BootstrapMainStarter());
    }

调用wrapperProperties方法找到当前目录下的gradle-wrapper.properties文件，该文件是用来指定一些配置信息的，后面会分析，先不管，接着会解析命令行传递过来的参数，最后一步是关键，会调用WrapperExecutor对象的execute方法，传入了一个Install对象，这个是用来处理gradle的下载，如果指定版本的gradle在指定的gradle仓库中没有，就先下载解压，execute方法传入的另一个对象是BootstrapMainStarter对象，这个是用来启动执行gradle的帮助类，最终会执行。接下来详细分析这两步的内部实现原理：

1. Install对象下载安装gradle库

		public File createDist(final WrapperConfiguration configuration) throws Exception {
	        final URI distributionUrl = configuration.getDistribution();
	        final String distributionSha256Sum = configuration.getDistributionSha256Sum();
	
	        final PathAssembler.LocalDistribution localDistribution = pathAssembler.getDistribution(configuration);
	        final File distDir = localDistribution.getDistributionDir();
	        final File localZipFile = localDistribution.getZipFile();
	
	        return exclusiveFileAccessManager.access(localZipFile, new Callable<File>() {
	            public File call() throws Exception {
	                final File markerFile = new File(localZipFile.getParentFile(), localZipFile.getName() + ".ok");
	                if (distDir.isDirectory() && markerFile.isFile()) {
	                    InstallCheck installCheck = verifyDistributionRoot(distDir, distDir.getAbsolutePath());
	                    if (installCheck.isVerified()) {
	                        return installCheck.gradleHome;
	                    }
	                    // Distribution is invalid. Try to reinstall.
	                    System.err.println(installCheck.failureMessage);
	                    markerFile.delete();
	                }
	
	                boolean needsDownload = !localZipFile.isFile();
	                URI safeDistributionUrl = Download.safeUri(distributionUrl);
	
	                if (needsDownload) {
	                    File tmpZipFile = new File(localZipFile.getParentFile(), localZipFile.getName() + ".part");
	                    tmpZipFile.delete();
	                    logger.log("Downloading " + safeDistributionUrl);
	                    download.download(distributionUrl, tmpZipFile);
	                    tmpZipFile.renameTo(localZipFile);
	                }
	
	                List<File> topLevelDirs = listDirs(distDir);
	                for (File dir : topLevelDirs) {
	                    logger.log("Deleting directory " + dir.getAbsolutePath());
	                    deleteDir(dir);
	                }
	
	                verifyDownloadChecksum(configuration.getDistribution().toString(), localZipFile, distributionSha256Sum);
	
	                try {
	                    unzip(localZipFile, distDir);
	                } catch (IOException e) {
	                    logger.log("Could not unzip " + localZipFile.getAbsolutePath() + " to " + distDir.getAbsolutePath() + ".");
	                    logger.log("Reason: " + e.getMessage());
	                    throw e;
	                }
	
	                InstallCheck installCheck = verifyDistributionRoot(distDir, safeDistributionUrl.toString());
	                if (installCheck.isVerified()) {
	                    setExecutablePermissions(installCheck.gradleHome);
	                    markerFile.createNewFile();
	                    return installCheck.gradleHome;
	                }
	                // Distribution couldn't be installed.
	                throw new RuntimeException(installCheck.failureMessage);
	            }
	        });
	    }

	代码比较长，但思路很清晰，主要是注意一些细节问题，有利于平时开发，主要步骤如下：

	（1）首先是拿到distributionUrl，这个就是gradle的下载地址，也就是我们在gradle-wrapper.properties文件中配置的那个链接。
	
	（2）然后调用ExclusiveFileAccessManager的access方法，这个方法主要是去lock一个lck文件，这个文件的作用就是实现在某个时刻只能有一个使用者在使用该gradle库，所以下载前需要lock住这个文件。

	（3）lock后，继续判断文件是否存在，如果已经存在了，就返回，否则就开始下载，下载存放目录是通过gradle-wrapper.properties文件的distributionPath属性配置的，下载时会先下载到一个part文件，等下载完成了，就重命名为最终的文件名，这里有个细节，每次下载都会删除掉之前的part文件，也就是说这里并不支持断点下载，如果文件比较大然后又因为网络原因下载失败了，下次还是得重新下载，可能又会继续失败，然后一直反反复复就会很烦，这里我们可以直接修改代码优化。

	（4）下载后，就要校验sha256，不过只有在gradle-wrapper.properties文件中配置了sha256码的情况下才会进行校验，没配置的就跳过。

	（5）接着是解压，因为下载下来是一个zip文件，解压后文件的存放目录是通过gradle-wrapper.properties文件的distributionPath属性配置的。

	（6）最后是简单校验解压后的文件是否正确，比如是否有gradle-launcher-xxx.jar之类。如果校验通过，还会创建一个ok文件，用于表示该gradle库下载校验通过，这个gradle库可以被正常使用。

	（7）ExclusiveFileAccessManager的access方法unlock前面那个lck文件，至此流程结束，该gradle可以被使用了。

	下载流程说完了，这里还有个问题，就是我们去看了一下下载后的目录，如下：

		C:\Users\mai\.gradle\wrapper\dists\gradle-5.4.1-all\3221gyojl5jsh0helicew7rwx

	我们发现这个路径跟distributionPath指定的还是有点不同，gradle-5.4.1-all目录还可以解释得通，不同版本肯定要放到不同目录下，所以干脆就用版本名命名目录就好了，但我们还看到下一级目录名为3221gyojl5jsh0helicew7rwx，这个就不太容易看得出来了，这时我们就去代码里查找：

	    public LocalDistribution getDistribution(WrapperConfiguration configuration) {
	        String baseName = getDistName(configuration.getDistribution());
	        String distName = removeExtension(baseName);
	        String rootDirName = rootDirName(distName, configuration);
	        File distDir = new File(getBaseDir(configuration.getDistributionBase()), configuration.getDistributionPath() + "/" + rootDirName);
	        File distZip = new File(getBaseDir(configuration.getZipBase()), configuration.getZipPath() + "/" + rootDirName + "/" + baseName);
	        return new LocalDistribution(distDir, distZip);
	    }
	
	    private String rootDirName(String distName, WrapperConfiguration configuration) {
	        String urlHash = getHash(Download.safeUri(configuration.getDistribution()).toString());
	        return distName + "/" + urlHash;
	    }

		private String getHash(String string) {
		        try {
		            MessageDigest messageDigest = MessageDigest.getInstance("MD5");
		            byte[] bytes = string.getBytes();
		            messageDigest.update(bytes);
		            return new BigInteger(1, messageDigest.digest()).toString(36);
		        } catch (Exception e) {
		            throw new RuntimeException("Could not hash input string.", e);
		        }
		}

	getHash方法的返回值就是这个3221gyojl5jsh0helicew7rwx目录名，可以看到它是计算传入的string的MD5值字节数组，然后转成BigInteger，最后用36进制表示返回字符串，而这个string的值，就是gradle的下载地址“https://services.gradle.org/distributions/gradle-5.4.1-all.zip”，我们可以自己调用getHash方法，传入这个地址，发现返回值就是3221gyojl5jsh0helicew7rwx，说明我们的分析是正确的。

	总结下，下载流程涉及到几个文件，lck文件用于实现使用互斥，part文件是下载过程中的临时文件，ok文件表示该gradle库可以正常使用，如果没有ok文件，执行gradlew命令后会导致重新解压（不存在zip文件就先下载）。

2. BootstrapMainStarter对象启动执行gradle库

	这个就比较简单了，BootstrapMainStarter的start方法：

		public void start(String[] args, File gradleHome) throws Exception {
	        File gradleJar = findLauncherJar(gradleHome);
	        if (gradleJar == null) {
	            throw new RuntimeException(String.format("Could not locate the Gradle launcher JAR in Gradle distribution '%s'.", gradleHome));
	        }
	        URLClassLoader contextClassLoader = new URLClassLoader(new URL[]{gradleJar.toURI().toURL()}, ClassLoader.getSystemClassLoader().getParent());
	        Thread.currentThread().setContextClassLoader(contextClassLoader);
	        Class<?> mainClass = contextClassLoader.loadClass("org.gradle.launcher.GradleMain");
	        Method mainMethod = mainClass.getMethod("main", String[].class);
	        mainMethod.invoke(null, new Object[]{args});
	        if (contextClassLoader instanceof Closeable) {
	            ((Closeable) contextClassLoader).close();
	        }
    	}
	
	可以看到，这里反射调用gradle-launcher-xxx.jar中的org.gradle.launcher.GradleMain的main方法，这个GradleMain类，也就是gradle的入口类，至于接下来的流程，后面的文章会继续分析。

总结，gradlew的作用就是在执行gradle前，确保gradle仓库中有项目所需要的gradle版本，如果没有就会先下载解压准备好。

## gradle-wrapper.properties文件

这个文件用来指定gradle-wrapper.jar执行所需要的配置信息，主要有五种信息：

	distributionBase=GRADLE_USER_HOME
	distributionPath=wrapper/dists
	zipStoreBase=GRADLE_USER_HOME
	zipStorePath=wrapper/dists
	distributionUrl=https\://services.gradle.org/distributions/gradle-5.4.1-all.zip

* distributionBase
	
	指定gradle库解压后存放目录的根目录，默认设置为GRADLE_USER_HOME，这个其实就是环境变量名，如果没有设定，就会采用默认的gradle仓库根目录，windows下为C:\Users\用户名\.gradle，如果要自己指定到某个目录，就去新建个叫GRADLE_USER_HOME的环境变量，然后路径填自己的路径就行，这段逻辑在GradleUserHomeLookup.java类里，有兴趣可以自己查看。

	distributionBase还有另外一个取值就是PROJECT，表示当前工程的根目录。

* distributionPath

	指定gradle库解压后存放目录的子目录。

* zipStoreBase
	
	指定下载的gradle压缩包的存放目录的根目录。

* zipStorePath

	指定下载的gradle压缩包的存放目录的子目录。

* distributionUrl

	指定gradle库的下载地址。

## 手动下载gradle

有时因为各种原因，执行gradlew后下载gradle比较慢，甚至失败，根据上面对gradlew下载流程的分析，我们可以自己去网上下载，然后按照规则存放文件，就可以直接使用了，比如说我们要下载gradle-5.6.4-all，具体的步骤如下：

1. 在C:\Users\mai\.gradle\wrapper\dists下新建目录gradle-5.6.4-all，进入目录。

2. 允许上面所说的getHash方法，传入https://services.gradle.org/distributions/gradle-5.6.4-all.zip，得到ankdp27end7byghfw1q2sw75f，以该字符串新建目录。

3. 去网上下载gradle-5.6.4-all.zip，放到ankdp27end7byghfw1q2sw75f目录下，解压。

4. 新建gradle-5.6.4-all.zip.lck和gradle-5.6.4-all.zip.ok两个文件。

这样就配置好了，我们把gradle-wrapper.properties的distributionUrl改为https\://services.gradle.org/distributions/gradle-5.6.4-all.zip，然后执行gradlew命令，发现已经检测到我们配置的gradle库了，不会再去下载。

## 参考资料

[https://github.com/gradle/gradle/tree/master/subprojects/wrapper](https://github.com/gradle/gradle/tree/master/subprojects/wrapper "https://github.com/gradle/gradle/tree/master/subprojects/wrapper")








