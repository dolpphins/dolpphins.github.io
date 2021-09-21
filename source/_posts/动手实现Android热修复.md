---
title: 动手实现Android热修复
date: 2020-11-21 22:09:29
tags:
---

### 一、代码修复

#### 1. 原理

原理是比较简单的，本质就是App在运行时，会有个dex数组，当要用到的class不存在时，就顺序从dex数组中寻找，只要找到就停止，因此可以把修复好的dex插入到这个dex数组最前面，那么找的时候就会找到修复好dex中的class，而不再是旧的有问题的class。

#### 2. 动手写代码

	public class App extends Application {
	
	    private static final String TAG = "App";
	
	    @Override
	    protected void attachBaseContext(Context base) {
	        super.attachBaseContext(base);
	        try {
	            ClassLoader classLoader = getClassLoader();
	            Log.i(TAG, "classLoader:" + classLoader);
	            Field pathListField = classLoader.getClass().getSuperclass().getDeclaredField("pathList");
	            pathListField.setAccessible(true);
	            Object pathList = pathListField.get(classLoader);
	            Field dexElementsField = pathList.getClass().getDeclaredField("dexElements");
	            dexElementsField.setAccessible(true);
	            Object[] dexElements = (Object[]) dexElementsField.get(pathList);
	
	            Log.i(TAG, "把补丁复制到私有目录（省略）");
	            File patchFile = getDir("patch", Context.MODE_PRIVATE);
	            if (!patchFile.isDirectory()) {
	                patchFile.mkdirs();
	            }
	            Log.i(TAG, patchFile.getAbsolutePath());
	            // 加载补丁
	            DexClassLoader dexClassLoader = new DexClassLoader(
	                    new File(patchFile, "patch.dex").getAbsolutePath(),
	                    "/sdcard/odex/", null, getClassLoader());
	            Object[] patchDexElements = hookDexElements(dexClassLoader);
	
	            // 合并
	            Object[] newDexElements = (Object[]) Array.newInstance(dexElements[0].getClass(), dexElements.length + patchDexElements.length);
	            int index = 0;
	            for (Object element : patchDexElements) {
	                newDexElements[index++] = element;
	            }
	            for (Object element : dexElements) {
	                newDexElements[index++] = element;
	            }
	
	            // 设置回去
	            dexElementsField.set(pathList, newDexElements);
	
	        } catch (Throwable t) {
	            t.printStackTrace();
	        }
	    }
	
	    private static Object[] hookDexElements(ClassLoader classLoader) {
	        try {
	            Field pathListField = classLoader.getClass().getSuperclass().getDeclaredField("pathList");
	            pathListField.setAccessible(true);
	            Object pathList = pathListField.get(classLoader);
	            Field dexElementsField = pathList.getClass().getDeclaredField("dexElements");
	            dexElementsField.setAccessible(true);
	            Object[] dexElements = (Object[]) dexElementsField.get(pathList);
	            return dexElements;
	        } catch (Throwable t) {
	            t.printStackTrace();
	        }
	        return null;
	    }
	}

这里分为几个步骤：

1. 先通过反射拿到dexElements数组。

2. 使用dx命令将class打包成补丁dex，如下：

		dx --dex --output=patch.dex ./

	注意要在包的最顶级目录中执行。

3. 有了补丁dex后，通过网络下载到私有目录下。

4. 使用DexClassLoader加载补丁dex，同样也拿到对应的dexElements数组。

5. 两个dexElements数组合并，补丁的在前面，原有的在后面。

6. 把合并好的新dexElements数组设置回ClassLoader中。

注意这里dex会有缓存，因此这个补丁操作必须尽早执行，如果已经用过某个class后再执行补丁操作，就不会生效了。

### 二、资源修复

#### 1. 原理

ActivityThread中有个ArrayMap<String, WeakReference>变量mPackages，保存包名和LoadedApk的键值对，一般都是只有一个，我们只需要变量这些键值对，拿到对于的LoadedApk对象，然后修改其中的mResDir字符串的值为新的资源包的路径，后面在创建AssetManager时就会指定为新的资源路径，也就实现了资源的替换。

#### 2. 动手写代码

	private void doPatchResources() {
	    try {
	        File patchResDir = getDir("patch_res", Context.MODE_PRIVATE);
	        if (!patchResDir.isDirectory()) {
	            patchResDir.mkdirs();
	        }
	        File patchResFile = new File(patchResDir, "fix.apk");
	        if (!patchResFile.isFile()) {
	            // 没有补丁资源
	            return;
	        }
	        // printFields();
	        // 获取ContextImpl对象
	        Field mBaseField = this.getClass().getSuperclass().getSuperclass().getDeclaredField("mBase");
	        mBaseField.setAccessible(true);
	        Object mBase = mBaseField.get(this);
	        Log.i(TAG, "mBase:" + mBase);
	        // 获取ActivityThread对象
	        Field mMainThreadField = mBase.getClass().getDeclaredField("mMainThread");
	        mMainThreadField.setAccessible(true);
	        Object mMainThread = mMainThreadField.get(mBase);
	        Log.i(TAG, "mMainThread:" + mMainThread);
	
	        // 获取mPackages
	        Field mPackagesField = mMainThread.getClass().getDeclaredField("mPackages");
	        mPackagesField.setAccessible(true);
	        ArrayMap<String, WeakReference<?>> mPackages = (ArrayMap<String, WeakReference<?>>) mPackagesField.get(mMainThread);
	        for (String key : mPackages.keySet()) {
	            Log.i(TAG, "key:" + key);
	            Object value = mPackages.get(key).get();
	            Log.i(TAG, "value:" + value);
	
	            Field mResDirField = value.getClass().getDeclaredField("mResDir");
	            mResDirField.setAccessible(true);
	            String mResDir = (String) mResDirField.get(value);
	            Log.i(TAG, "mResDir:" + mResDir);
	            // 修改mResDir的值
	            String newPath = patchResFile.getAbsolutePath();
	            mResDirField.set(value, newPath);
	        }
	    } catch (Throwable t) {
	        t.printStackTrace();
	    }
	}

这里也是分为几个步骤：

1. 打包好新的资源包fix.apk，手动放到私有目录下。

2. 拿到Application对象的ContextImpl对象，再拿到其中的ActivityThread对象，一个进程对应有一个。

3. 拿到ActivityThread对象中的mPackages，也就是ArrayMap<String, WeakReference>表。

4. 遍历该表，获取到LoadedApk对象，修改其中的mResDir为fix.apk路径。

5. 这样后面在创建AssetManager对象时，资源路径就是这个fix.apk的路径，获取到的资源也就是新的。