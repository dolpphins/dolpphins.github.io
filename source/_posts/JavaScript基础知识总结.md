---
title: JavaScript基础知识总结
date: 2020-08-01 22:09:29
tags:
---

### 一、常见数据类型

JavaScript中可以用typeof获取某个变量的数据类型，常见的数据类型有：number，string，boolean，object，function，undefined，直接看代码：

    let a = 1;
	let b = 1.222;
	alert(typeof(a)); // number
	alert(typeof(b)); // number
	
	let c = '1';
	let d = 'hello';
	let k = '';
	alert(typeof(c)); // string
	alert(typeof(d)); // string
	alert(typeof(k)); // string
	
	let e = true;
	alert(typeof(e)); // boolean
	
	let m = window;
	let n = {};
	let p = null;
	let q = [1, 2, 3];
	alert(typeof(m)); // object
	alert(typeof(n)); // object
	alert(typeof(p)); // object
	alert(typeof(q)); // object
	
	let fun = function() {}
	alert(typeof(fun)); // function
	
	let json = {}
	alert(json.a); // undefined
	let y;
	alert(y); // undefined
	let fun1 = function(){
		return;
	}
	alert(fun1()); // undefined
	let fun2 = function(){}
	alert(fun2()); // undefined

### 二、类型转换

#### 1. 显式类型转换

通过全局函数parseInt，parseFloat进行转换，如下：

    let a = '123'
	alert(parseInt(a) + 1); // 124

parseInt会从左到右扫描，直到遇到无法解析字符为止，如果第一个字符就无法解析，那么就返回NaN，如下：

	let b = '123hello'
	alert(parseInt(b)); // 123
	let c = '1hello23'
	alert(parseInt(c)); // 1
	let d = '2.333'
	alert(parseInt(d)); // 2
	let e = 'hello'
	alert(parseInt(e)); // NaN

注意NaN不能用==进行判断相等，需要用isNaN函数进行判断。

#### 2. 隐式类型转换

	let a = 123;
	let b = 123;
	let c = '123hello';
	alert(a == b); // true
	alert(a == c); // false

这里b会被先转换成number类型123，然后再与a进行比较，结果返回true，而c就不会，所以返回false，如果需要判断类型可以改为用===，这时123就不会和’123’相等了。

在使用+，-等数学运算符时也可能存在隐式类型转换：

	let a = '10';
	let b = 2;
	alert(a - b); // 8
	let c = '10hello';
	alert(c - b); // NaN
	let d = '3';
	alert(a + b); // '102'

对于能直接字符串连接的就直接连接，否则就尝试转换成number再运算，对于无法转换的返回NaN。

### 三、变量作用域

子函数可以使用父函数定义的局部变量。

### 四、数组

JavaScript中定义数组有两种方式：

	let arr1 = [1, 2, 3, 4];
	let arr2 = new Array(5, 6, 7);
	alert(arr1);
	alert(arr2);

而且同个数组可以存放不同类型数据：

	let arr3 = [1, 'hello', 2];
	alert(arr3);

数组有个length属性，表示数组的长度，而且该属性值还可以修改：

	let arr1 = [1, 2, 3, 4];
	alert(arr1.length); // 4
	arr1.length = 2;
	alert(arr1); // [1, 2]

数组本身就是对象，有一系列的函数可以调用，比如在数组尾部添加和删除元素用push和pop：

	let arr1 = [1, 2, 3, 4];
	arr1.push(5);
	alert(arr1); // [1, 2, 3, 4, 5]
	arr1.pop();
	arr1.pop();
	alert(arr1); // [1, 2, 3]

相对应的，在数组头添加和删除元素用unshift和shift：

	let arr1 = [1, 2, 3, 4];
	arr1.unshift(0);
	alert(arr1); // [0, 1, 2, 3, 4]
	arr1.shift();
	arr1.shift();
	alert(arr1); // [2, 3, 4]

另外还有个splice函数，用于在指定位置删除和添加元素：

	let arr1 = [1, 2, 3, 4];
	// (起始位置，要删除元素个数，要添加的元素列表)
	arr1.splice(1, 2, 'a', 'b');
	alert(arr1); // [1, a, b, 4]

concat函数用于连接多个数组，join函数用于数组转字符串：

	let arr1 = [1, 2, 3, 4];
	let arr2 = arr1.concat([1, 2, 3]);
	let arr3 = arr1.concat(1);
	alert(arr1); // 原数组不会改变，[1, 2, 3, 4]
	alert(arr2); // 连接后得到的副本，[1, 2, 3, 4, 1, 2, 3]
	alert(arr3); // [1, 2, 3, 4, 1]
	
	let arr1 = [1, 2, 3, 4];
	alert(arr1.join('-')); // 1-2-3-4

最后一个是数组排序函数sort，注意它默认都是按照字符串进行排序的，即使数组中的元素都是number类型：

	let arr = [8, 9, 10, 11, 3, 2];
	arr.sort();
	alert(arr); // [10, 11, 2, 3, 8, 9]

如果要按照数值大小进行排序，需要传入比较函数：

	let arr = [8, 9, 10, 11, 3, 2];
	arr.sort(function(n1, n2){
		return n1 - n2;
	});
	alert(arr); // [2, 3, 8, 9, 10, 11]

### 五、条件和循环

这两个和C++中的类似，都是if，else，switch，for，while，break，continue这些，另外JavaScript也有三目表达式。需要注意的就是JavaScript中可以用for…in进行循环遍历：

	let arr = [1, 2, 3];
	for (let index in arr) {
		alert(arr[index]);
	}

这里需要注意条件语句中的判断条件为false的情况：

	let a; 
	alert(a ? 'true' : 'false'); // undefined false
	let b = null;
	alert(b ? 'true' : 'false'); // 空对象 false
	let c = '';
	alert(c ? 'true' : 'false'); // 空字符串 false
	let d = 0;
	alert(d ? 'true' : 'false'); // 0 false

也就是说，这四种情况下都是判断为false，其它情况判断为true。

### 六、Json

JavaScript中经常会使用到json对象，比如请求接口时后端的返回结果等，访问json中的值可以用.或者[]：

	let json = {a: 1, b: 2, c: 3};
	alert(json.a); // 1
	alert(json['b']); // 2
	alert(json.d); // undefined

遍历json：

	let json = {a: 1, b: 2, c: 3};
	for (let key in json) {
		alert(json[key]);
	}

### 七、函数

JavaScript中没有真正意义上的函数重载，但对于某个函数，参数个数是不固定的，可以通过arguments获取到：

	let fun = function() {
		alert(arguments.length); // 2
	}
	fun(1, 2);

也就是说，调用函数时不能看函数参数签名，参数签名列表可以是空，但外部还是可以传参数进去的，内部也可以通过arguments获取到对于的参数，不过为了可读性，一般都会写上参数列表：

	let fun = function(a, b, c) {
		alert(a + b + c);
	}
	fun(1, 2, 3);

### 八、事件队列

avaScript只有一个线程在干活，不支持多线程，但它又是非阻塞的，其内部使用了事件队列。

JavaScript中有两个事件队列：宏任务队列和微任务队列，setTimeout和setInterval等任务会被加到宏任务队列中，Promise等任务会被加到微任务队列中，在优先级上：主线程>微任务队列>宏任务队列。可以通过下面的示例代码理解这个机制：

	console.log('start');
	setTimeout(function() {
	    console.log('222');
	}，0)
	new Promise((resolve, reject) => {
	    console.log('333');
	    resolve();
	    console.log('444');
	}).then(value => {
	    console.log('success');
	}, reason => {
	    console.log('fail');
	});
	console.log('111');

上面代码被执行，首先肯定是先输出start，然后执行了setTimeout函数，这时会把任务加到宏任务队列中，接着又创建了一个Promise对象然后执行其任务，输出333，回调resolve，这时会把任务加到微任务队列中，然后输出444，继续往下执行，输出111，这时主线程的代码都执行完了，就开始轮询事件队列，先取出微任务队列中的所有任务，这里面有刚才的resolve回调，开始执行，输出success，微任务队列的任务都执行完了，继续取宏任务队列中的队头任务，是刚才setTimeout的任务，因此输出222。

所以输出的日志为：start–>333–>444->111->success–>222
