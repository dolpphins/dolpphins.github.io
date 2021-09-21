---
title: Javascript中的async和await
date: 2021-01-23 22:09:29
tags:
---

在ES7中async和await被正式发布，它们是基于Promise的语法糖，可以用于优化链式代码的写法，让代码更具有可读性，编写逻辑更加清晰。

#### async

async只需要记住两点就可以：

1. async函数最终都将返回一个Promise对象，返回值通过resolve的参数传递。

	比如这段代码：

		async function test() {
		    return "hello js"
		}

	实际上等价于：

		async function test() {
		    return Promise.resolve("hello js");
		}

	因此在调用test函数时，可以在then中获取到结果：

		test().then(value => {
		  console.log(value);
		})

	所以所有的async函数都不会阻塞当前后面代码的执行，结果需要在then中获取。

2. await所在的函数必须加上async。

	这个是语法规定的，没什么好说。

#### await

await可以是表达式，普通函数，或者Promise对象，当表达式计算完成了，普通函数执行结束了，Promise对象resolve了，那么await就等到结果了，继续往下执行async函数内接下来的代码，而async函数外的代码，在await时就会被执行。

举个例子：
	
	function a() {
	  return new Promise((resolve, reject) => {
	    for (let i = 0; i < 100000; i++) {
	      for (let j = 0; j < 100000; j++) {}
	    }
	    resolve('haha');
	  });
	}
	
	async function b() {
	    let result = await a();
	    console.log(result);
	}
	
	b();
	console.log('22222');

在函数b里，await了函数a，当Promise对象resolve之后，才继续往下执行代码，打印出haha，而在await的同时，外部代码继续执行，打印2222，因此2222会在haha之前打印。

可以使用await减少then链的出现，如果上面的例子不用await，又要打印haha，那么就需要这样子写：

	async function b() {
	    a().then(value => {
	      console.log(value);
	    })
	}

很明显，这种写法可读性没有上面那种好，如果有很多then，那可读性就更差了。