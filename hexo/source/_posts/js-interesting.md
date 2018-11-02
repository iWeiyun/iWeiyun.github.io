---
title: 不一样的JavaScript
author: matthew
date: 2018-10-15 22:30:08
tags: js,javascript
---



js语法看起来是类c的，如果有c语言基础，可以看懂一些js代码，尤其是类似这样的代码：

```js
for (i = 0; i < 10; i++) {
    // code
}
```

只看这段代码，跟c的写法甚至完全一样。很容易让你有种错觉，简单看下js语法，就能写出优秀的js代码。



但是，不要被骗了，当你看到后面的代码时，就不会再这样想了。下面就列举一些js语法上感觉有趣或比较特别的例子

​	



----

​	



##### 变量声明可以放在使用之后

```javascript
function foo() {
    a = 3;			// 此处使用
    console.log(a);
    var a;			// 此处声明
}
```

​	



-----

​	



##### 对象可以动态的增加属性，不需要提前声明

```javascript
var o = {};
o.p1 = "good";
o.p2 = 35;
console.log(o);	// Object {p1: "good", p2: 35}
```

​	



------

​	



##### 对象的属性也可地动态的删除

```javascript
var o = {p1: "good", p2: 35};
delete o.p1;
console.log(o);	// Object {p2: 35}
```

​	



-----

​	



##### 函数也是对象，也有方法。如可以调用length，获取参数个数

```javascript
function myAdd(a, b) {
    return a + b;
}
console.log(myAdd.length);	// 输出2
```

也是支持自定义的方法添加的

​	



-------

​	



##### 函数定义时，可以不指定参数，但在使用时可以传任意参数

```javascript
function noParamFunc() {
    console.log(arguments)
}
noParamFunc(1, 2)	// 输出1, 2
```

​	



-----

​	



##### 可以动态决定函数的定义

```javascript
if (condition) {
    function sayHi() {
        console.log("Hi");
    }
} else {
    function sayHi() {
        console.log("Hey");
    }
}
sayHi();		// 根据condition的值输出Hi或Hey
```

​	



-----

​	



##### 函数可以这样定义

```javascript
var myAdd2 = x => x + 2;
myAdd2(5);
```

这种写法在定义简短的函数以及闭包时会非常简洁

如写出如下简洁实用的函数：

```javascript
const longestConsec = (a, k) => (k < 1 || a.length < 1 || k > a.length)
? ''
: a.map((_, i, a) => a.slice(i, i + k).join(''))
.reduce((a, b) => a.length >= b.length ? a : b)
```

​	



-----

​	



##### 函数定义时将其赋值给其它变量，则函数名在外部不再可用

```javascript
var foo = function bar() {
    console.log('good');
}
bar();  // 出错，这儿只能通过foo()来调用
foo();  // ok
```

但是，如下的写法又是正确的：

```javascript
function bar() {
    console.log('good');
}
var foo = bar;
bar();  // ok
foo();  // ok
```

​	



-----

​	



##### 在for循环里定义的变量，作用域是超出for的

```javascript
for (var i = 0; i < 10; i++) {
    console.log(i);
}
console.log(i); // 仍然可以访问，且是访问的上面的这个i
```

​	



-----

​	



##### 一元加号，可以将字符串转为数字

```javascript
+"10"		// 就是数值10
```

​	



-----

​	



##### 数组创建的歧义

```javascript
// 看起来相同的写法，但行为预期却不一样
var a1 = new Array(3, 4, 5);
var a2 = new Array(3);
console.log(a1);     // [3, 4, 5]
console.log(a2);     // [] 长度为3的空数组
```

​	



-----

​	



##### 作用域不注意，可能行为也是不可预期

```javascript
// 全局作用域
for(var i = 0; i < 10; i++) {
    subLoop();
}

function subLoop() {
    for(i = 0; i < 10; i++) {
        console.log(i);
    }
}
```

这段代码本来是想执行100次的循环，但实际只会执行10次

​	



-----

​	



##### 奇怪的相等判断

```javascript
0 	==	""            // true
0	==	"0"           // true
"" 	== 	"0"           // false
```

​	



-----

​	



##### 奇怪但有效的表达式

```javascript
[1,2,3][1,2];	// 输出3
```

```javascript
++[[]][+[]]+[+[]];
```

这个也是有效的表达式，表达式的值是10。 具体原因可以参考[这篇文章](http://justjavac.com/javascript/2012/05/24/can-you-explain-why-10.html)

​	



-----

​	



##### JS语法本身是需要分号的

```javascript
var a = 5
var b = 6
// code
```

虽然写的代码可以不加分号，在解释执行时，解释器会帮我们补上分号：

```javascript
var a = 5;
var b = 6;
// code
```

但有时候依赖于解释器加分号的话，行为可能不是预期的

​	



-----

​	



**参考资料**

[JavaScript秘密花园](https://bonsaiden.github.io/JavaScript-Garden/zh)

[JavaScript之Function函数深入总结](https://www.cnblogs.com/venoral/p/5280805.html)