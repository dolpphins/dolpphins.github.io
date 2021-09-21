---
title: 深入分析Promise
date: 2020-09-26 22:09:29
tags:
---

Promise用于解决代码嵌套问题，在js中被大量使用，对于如何解决代码嵌套，可以参考之前的文章《使用Promise解决代码嵌套问题》。

要做到熟悉掌握Promise，还需要弄清一些常见的问题，接下来逐一分析。

### 一、问题：什么时候会执行catch分支？

#### 先从最简单的开始，只有一个Promise函数：

	p().then(value => {
	    console.log('111');
	}, reason => {
	    console.log(reason);
	}).catch(e => {
	    console.log('crash:' + e);
	});
	
	function p() {
	    return new Promise((resolve, reject) => {
	        throw 'hahaha'            
	    });
	}

代码最终会输出：hahaha，也就是说走到了reject分支，catch分支并没有走到，这时我们把reject分支去掉，如下：

	p().then(value => {
	    console.log('111');
	}).catch(e => {
	    console.log('crash:' + e);
	});

这种情况就会输出：crash:hahaha，走到了catch分支里，这时如果改成去掉catch分支，加上reject分支，结果会输出什么呢？其实这里还是会走到reject分支里。

那如果现在要求存在reject，又要走到catch分支，要怎么实现呢？可以在reject里继续抛出异常：

	p().then(value => {
	    console.log('111');
	}, reason => {
	    console.log(reason);
	    throw reason;
	}).catch(e => {
	    console.log('crash:' + e);
	});

这时就会输出：

	hahaha
	crash:hahaha

#### 接着加大难度，在Promise代码链中抛异常：

	function a() {
	  console.log('执行a');
	    return new Promise((resolve, reject) => {
	        throw 'hahaha';
	    });
	}
	
	function b() {
	  console.log('执行b');
	  return new Promise((resolve, reject) => {
	        resolve();
	    });
	}
	
	function c() {
	  console.log('执行c');
	  return new Promise((resolve, reject) => {
	        resolve();
	    });
	}
	
	a().then(value => {
	    return b();
	}).then(value => {
	    return c();
	}).then(value => {
	    console.log('最终结果成功');
	}, reason => {
	    console.log('最终结果失败');
	});

函数a抛异常了，因为后面两个then都没有reject，因此直接进入第三个then的reject，最终打印结果：
	
	执行a
	最终结果失败

假如把第三个then的reject去掉，后面再加上catch，如下：

	a().then(value => {
	    return b();
	}).then(value => {
	    return c();
	}).then(value => {
	    console.log('最终结果成功');
	}).catch(e => {
	    console.log('crash:' + e);
	});

意料之中，执行到了catch分支中，打印结果：

	执行a
	crash:hahaha

#### 再加大难度，catch放到Promise链的中间，如下：

	function a() {
	  console.log('执行a');
	    return new Promise((resolve, reject) => {
	        resolve();
	    });
	}
	
	function b() {
	  console.log('执行b');
	  return new Promise((resolve, reject) => {
	        resolve();
	    });
	}
	
	function c() {
	  console.log('执行c');
	  return new Promise((resolve, reject) => {
	        resolve();
	    });
	}
	
	function d() {
	  console.log('执行d');
	  return new Promise((resolve, reject) => {
	        resolve();
	    });
	}
	
	a().then(value => {
	    return b();
	}).then(value => {
	    return c();
	}).catch(e => {
	    console.log('crash:' + e);
	}).then(value => {
	    return d();
	}).then(value => {
	    console.log('最终结果成功');
	}, reason => {
	    console.log('最终结果失败');
	});

针对上面的代码：

1. 当a()执行成功，回调resolve时，catch分支不会被执行。
2. 当a()执行失败，回调reject时，catch分支被执行，后面又从resolve开始执行。
3. 当a()执行抛出异常时，catch分支被执行，后面又从resolve开始执行。

**总结：在Promise链中，当发生了reject或者carsh，会往后查找第一个reject或者catch分支，有就执行，然后接着恢复到resolve状态，继续往下执行，如果没有找到，就crash报错。**

### 二、问题：Promise.all的作用和用法？

Promise提供了all函数，用于同时执行多个Promise任务，代码如下：

	function a() {
	    return new Promise((resolve, reject) => {
	        console.log('a执行中...');
	        resolve('a成功');
	    });
	}
	
	function b() {
	  return new Promise((resolve, reject) => {
	        console.log('b执行中...');
	        resolve('b成功');
	    });
	}
	
	function c() {
	  console.log('执行c');
	  return new Promise((resolve, reject) => {
	        console.log('c执行中...');
	        resolve('c成功');
	    });
	}
	
	Promise.all([a(), b(), c()]).then(values => {
	    console.log(values);
	}, reason => {
	    console.log(reason);
	});

a函数，b函数和c函数都会被同时触发执行，如果有一个函数reject了，就马上回调到then的reject中，参数reason就是第一个reject传过来的，注意这时其它函数还会继续执行，而如果三个最终都resolve了，这时就会回调到then的resolve中，values就是参数数组。

### 三、问题：Promise.race的作用和用法？

与Promise.all类似，区别就是对于Promise.race，只要有一个Promise任务resolve了，就马上回调到then的resolve中，注意这时其它的Promise任务还是会继续执行的，而它们最后是resolve还是reject都会被忽略了。

很明显，对于Promise.race，then的resolve参数只有一个value，而不是values。
