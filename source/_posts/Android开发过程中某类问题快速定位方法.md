---
title: Android开发过程中某类问题快速定位方法
date: 2021-09-21 21:22:33
tags:
---

### 一、问题

测试那边反馈有问题，自测没问题，由于各种原因无法拿测试手机亲自调试，因此只能是测试复现问题，然后给日志，开发根据日志定位问题原因。

前提：代码逻辑复杂，流程复杂，或者是刚接手过来的项目，如何快速地定位到整个流程中哪一步出了问题？

### 二、分析

方案一：叫测试给指定tag的日志，因为代码逻辑复杂类多，很难一次性给出所有的tag，后面发现日志缺失后再去叫测试补日志很浪费时间。

方案二：叫测试给全量日志，日志会很多，然后在全量日志里去搜索某条日志，可能会找出很多条，位置也不同，跳来跳去很容易看晕。

### 三、解决

测试给全量日志，然后编写工具脚本，对全量日志进行逐行对比，如果包括指定的关键字，就输出，实现日志自动过滤功能。

然后根据逻辑，要看某个日志有没有打印，就指定日志关键字，然后执行工具脚本代码，就可以马上看到该行日志，而不需要去全量日志里面查找。这样就能一条一条日志地查找，确定代码执行逻辑，再通过日志内容，就可以确定整个流程中的哪个步骤出了问题。

### 四、总结

遇到类似的问题，就可以分几个步骤快速进行：

1.  测试给全量日志。
2. 添加要查看的日志关键字，执行工具脚本代码。
3. 查看是否有日志以及日志内容。
4. 重复2和3，直到找出数据异常的代码逻辑位置。

### 五、附录（工具脚本代码）

	public class LogKeyCutter {
	
		private static final String srcPath = "/Users/cat/Library/Containers/rt3.txt";
		
		private static final Set<String> sLogKeywords;
		
		static {
			sLogKeywords = new HashSet<String>();
			sLogKeywords.add("getGameGiftCacheUpdateTime");
		}
		
		public static void main(String[] args) {
			List<String> lines = readTxt(srcPath);
			for (String str : lines) {
				for (String keyword : sLogKeywords) {
					if (str.contains(keyword)) {
						System.out.println(str);
						break;
					}
				}
			}
		}
	}
	
	public static List<String> readTxt(String path) {
		List<String> result = new ArrayList<String>();
		BufferedReader br = null;
		try {
			br = new BufferedReader(new InputStreamReader(new FileInputStream(path), Charset.forName("UTF-8")));
			String line = null;
			while ((line = br.readLine()) != null) {
				try {
					result.add(line);
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			closeIo(br);
		}
		return result;
	}
