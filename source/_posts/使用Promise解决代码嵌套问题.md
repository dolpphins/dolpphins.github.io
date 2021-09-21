---
title: 使用Promise解决代码嵌套问题
date: 2020-09-19 22:09:29
tags:
---

### 一、问题

现在有多个请求，第一个请求结束后，失败就结束，成功就继续执行第二个请求，第二个请求结束后，失败就结束，成功就继续执行第三个请求，……，这样一直下去，可能会有三个请求，或者五个请求，或者更多。

### 二、原有的解决方法

按照以往的思路，会写出如下的嵌套代码：

	a(function(res) {
	    if (res) {
	        console.log('a执行成功');
	        b(function(res) {
	            if (res) {
	                console.log('b执行成功');
	                c(function(res) {
	                    if (res) {
	                        console.log('c执行成功');
	                    } else {
	                        console.log('c执行失败，结束');
	                    }
	                });
	            } else {
	                console.log('b执行失败，结束');
	            }
	        });
	    } else {
	        console.log('a执行失败，结束');
	    }
	});
	
	function a(callback) {
	  callback(true);
	}
	
	function b(callback) {
	  callback(true);
	}
	
	function c(callback) {
	  callback(true);
	}

### 三、使用Promise解决嵌套回调

如果请求数再多一点，那嵌套会更加严重，可读性很差，这时可以用Promise进行优化，如下：

	a().then(value => {
	   console.log(value);
	   return b();
	}, reason => {
	   console.log(reason);
	   return b();
	}).then(value => {
	   console.log(value);
	   return c();
	}, reason => {
	   console.log(reason);
	   return c();
	}).then(value => {
	   console.log(value);
	}, reason => {
	   console.log(reason);
	});
	
	function a() {
	    return new Promise((resolve, reject) => {
	      if (true) {
	          resolve('a执行成功');
	      } else {
	        reject('a执行失败，结束');
	      }
	    })
	}
	
	function b() {
	  return new Promise((resolve, reject) => {
	      if (true) {
	        resolve('b执行成功');
	      } else {
	        reject('b执行失败，结束');
	      }
	    })
	}
	
	function c() {
	  return new Promise((resolve, reject) => {
	      if (true) {
	        resolve('c执行成功');
	      } else {
	        reject('c执行失败，结束');
	      }
	    })
	}

每个函数都不再采用callback进行回调，而是改为返回Promise对象，外部就可以在then里面获取到执行结果，然后继续调用下一个函数，还是返回Promise对象，在下一个then里获取到。

**这里的关键就是：then里面返回的结果，会在下一个then里拿到，这里所说的结果，包括某个值，或者是(resolve, reject)这样的回调。**

### 四、如何提前终止Promise链？

上面的代码还是有问题的，如果在中间某个环节出现了reject，它只会导致下一个then执行到reject分支，下下个then又是执行了resolve分支，很明显，我们期望是只要有一个reject了，后续都要执行reject分支。

这个问题的原因是因为在于我们处理了reject分支，但reject里又没有返回新的Promise，导致下个then进入resolve分支，这里解决的方法有两个：

1. 改成都返回一个reject状态的Promise对象。
2. 直接把前面的reject分支都去掉，最后面的then才处理reject。

### 五、如何实现延时有序执行？

如果b()需要延时5s后执行，要怎么写？显然不能写成这样：

	setTimeout(function(){
	    return b();
	}, 5000);

etTimeout里面的任务会放到宏任务队列，而Promise的任务会被放到微任务队列，两者之间没有执行顺序约束，而我们是要在b执行后才能执行c，所以并不能实现我们的目的，**这里的解决方法，就是要保证setTimeout里面的任务执行结束，才能继续执行接下来的Promise任务，可以引入一个中间Promise实现，定义delay函数：**

	function delay(delayTime) {
	    return new Promise((resolve, reject) => {
	        setTimeout(function() {
	          resolve();
	        }, delayTime);
	    });
	}

这里在调用delay函数后，当超时时间到了，setTimeout里的任务执行了，才会回调resolve，然后才继续执行接下来的then里面的代码。

### 六、最终Promise代码

解决了上面两个常见问题，最终的代码如下：

	hello().then(value => {
	    console.log('成功');
	}, reason => {
	    console.log('失败');
	});
	
	function hello() {
	  return new Promise((resolve, reject) => {
	      console.log('执行a');
	      a().then(
	        value => {
	            return delay(2000);
	        }, reason => Promise.reject(reason)).then(value => {
	            console.log('执行b');
	            return b();
	      }, reason => Promise.reject(reason)).then(value => {
	            console.log('执行c');
	            return c();
	      }, reason => Promise.reject(reason)).then(value => {
	            console.log('执行d');
	            return d();
	      }, reason => Promise.reject(reason)).then(
	        resolve, reject
	      );
	  });
	}
	
	function a() {
	    return new Promise((resolve, reject) => {
	      if (true) {
	          resolve('a执行成功');
	      } else {
	        reject('a执行失败，结束');
	      }
	    })
	}
	
	function b() {
	  return new Promise((resolve, reject) => {
	      if (false) {
	        resolve('b执行成功');
	      } else {
	        reject('b执行失败，结束');
	      }
	    })
	}
	
	function c() {
	  return new Promise((resolve, reject) => {
	      if (true) {
	        resolve('c执行成功');
	      } else {
	        reject('c执行失败，结束');
	      }
	    })
	}
	
	function d() {
	  return new Promise((resolve, reject) => {
	      if (true) {
	        resolve('d执行成功');
	      } else {
	        reject('d执行失败，结束');
	      }
	    })
	}
	
	function delay(delayTime) {
	    return new Promise((resolve, reject) => {
	        setTimeout(function() {
	          resolve();
	        }, delayTime);
	    });
	}