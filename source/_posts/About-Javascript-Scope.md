---
title: 关于 JavaScript 的作用域
date: 2017-11-09 11:43:50
tags:
---
最近写了点 React 的东西，因为脑子里面一直都是 Objective-C 和 Swift，所以写 JS 的时候总感觉哪里不对，虽然用的是 ES6 感受上好了很多。然后翻了下 MDN 和 You Don't Know JS 这本书，才发现按照往常的思维来理解 JS 代码是会出大乱子的，跟我所认为的完全不一样。索性按照教程和书上的例子，来开始写写针对 JS 的一些总结，这里先讲讲作用域。
<!-- more -->

在 js 中声明一个变量，ES6 之前的方法是用 var 或者不用 var, 区别在于不用 var 的时候这个变量的作用域是全局（node 中为 global 而浏览器中则为 window），但是，如果你用 var 来声明，那么不代表这个变量就存在与你想象中的那个作用域中。比如下面这段代码：

```js
function foo() {
	var i = 0;
	if (1) {
		var j = 0;
		for (var k = 0; k < 3; k++) {
		    //1
			console.log('before k is:' + k);
		}
		//2
		console.log('after k is:' + k);
	}
	//3
	console.log('j is:' + j);
	console.log('k is:' + k);
}
foo();
```

如果用 Swift 的思维逻辑来分析这段代码的话，在 1 中会返回想象中的那样，然后 2、3 的代码因为访问的变量存在 if 的作用域中，所以外部访问不到，直接报错。按照这样想的话，一切很合乎常理，**但是**，在 Javascript 中一切都那么不一样。下面是 js 代码的输出结果：

```js
before k is:0
before k is:1
before k is:2
after k is:3
j is:0
k is:3
```

在 JS 中，以 var 声明的变量不是按照 {...} 来当作作用域单元的，所以在上面的代码中 if 语句之外还能访问到 if 语句内用 var 声明的变量。我觉得可以理解为用 var 声明的变量是以 function {...} 来作为作用域单元。按照这样的逻辑来思考下面的代码输出：

```js
var foo = true, baz = 10;
if (foo) {
	var baz = 3;
	console.log(baz);
}
console.log(baz);

// 输出为
// 3
// 3
```

前面提到了 function，在 js 中，如果一个语句以 function 开头那一般可以将其理解为这就是函数声明，否则的话这就是一个函数表达式。还是以代码举例子：

```js
var a = 2;
// 1
function foo() {
	var a = 3;
	console.log(a);
}
foo();
console.log(a);
```

这段代码中，注释 1 下面的语句为函数声明，声明了一个 foo 的函数，然后函数内为一个独立的作用域，所以再次作用域声明了变量 a 并赋值，又在此作用域进行打印自然就是该作用域下的值。而在最后一句又对 a 进行打印，这次的 a 是全局中的 a，所以执行这段代码输出的分别为 3 和 2.

```js
function foo() {            (function foo() {
	var a = 3;                  var a = 3; 
	console.log(a);             console.log(a);
}                           })();
foo();                            
```

上面两个代码的输出其实是一样的，但是不同在于，左边的声明的函数 foo 被绑定在全局作用域中，所以直接可以在全局作用域进行调用；而右边的 foo 被当定在函数表达式自身的函数当中，即 (function foo {...})() 的作用域为 ... 所在的作用域。

紧接着，再来介绍“提升”这个概念，先直接看代码。

```js
a = 2;
var a;
console.log(a);

console.log(b);
var b = 2;
```

按照正常理解，先声明了变量 a 并赋值，然后又让 a 的赋值为空，至于 b ，在一开始打印的时候都没有声明 b，那肯定会直接报错。感觉这种理解没什么问题，**但是这是错的**，实际上这段 js 代码执行后的结果是打印 a 为 2，打印 b 为 underfined. 因为在 js 中，编译器会先找到所有的声明，然后使用合适的作用域将其进行关联，就是编译时会先关心声明再进行赋值。所以，这一行为，也就是变量和函数声明会从它们在代码中出现的位置被移动到了最上面，叫做提升。也就是说，上面这个对 b 进行操作的代码，相当于下面这样：

```js
var b;
console.log(b);
b = 2;
```

所以这样也就理解为什么 b 的输出是 undefined 了。那其实下面这段代码也很好理解：

```js
foo()
var foo = function () {
	//...
}
// TypeError undefine is not a function
```

一开始对 foo 进行了函数提升，所以 foo 为 undefined，但是 undefined 并不是一个 function，所以对 undefined 执行 () 这样的操作就会报错。这下应该好理解提升的概念了，在提升中，当有函数和变量出现重复声明的时候，函数会优先被提升，但是如果有两个重复声明的函数出现提升时，后声明的函数会覆盖之前声明的函数。按照这样的原则，下面的代码中函数被优先提升，所以输出为 1：

```js
foo();
var foo;
function foo() {
	console.log(1);
}
foo = function () {
	console.log(2);
}
```

最后讲一下 js 的闭包，这个其实了解 Swift 闭包的话就很好理解这个概念，我所理解的闭包其实就是一个自闭和函数，或者说当你在将一个函数当作第一级的值并进行传递，其实就是在使用闭包了。比如下面这一个经典的代码：

```c
function foo() {
	var a = 2;
	
	function bar() {
		console.log(a);
	}

	return bar;
}
var x = foo();
x();
```

在这段代码中，函数 bar 能够访问到 foo() 的内部作用域，然后将 bar 函数本身进行返回，也相当于对外部做传递，使得外部可以访问到 bar 所在的作用域中的变量。所以，当一个函数可以记住并访问所在的词法作用域，即使函数是在当前词法作用域之外执行，这时就产生了闭包。理解了这一概念之后，我们再看一个经典的代码：

```c
for (var i = 0; i <= 5; i++) {
	setTimeout(function timer() {
		console.log(i);
	}, i*1000);
}
```

在上面这个代码中，我们期望是能在不同的循环迭代中输出不同的且递增的 i 值，但实际上我们最后输出的都是一样的值。这是因为虽然在不同迭代中都会去定义一个 timer，但是他们都属于同一个共享的作用域中，而在这一作用域中只有一个 i，所以最后访问到的 i 都是一个值。那既然这样，我是不是按照上面的一个思路来这样改就行了？

```c
for (var i = 0; i <= 5; i++) {
	(function () {
		setTimeout(function timer() {
				console.log(i);
		}, i*1000);
	})();
}
```

我在上面的代码中，让 timer 拥有自己的作用域，但事实上这个代码运行的结果还是一样，因为虽然对 timer 增加了作用域，但是这个增加的作用域中并没有 i，或者说在这个作用域中访问的 i 还是外部的 i，所以在此基础上再来做一些改变：

```c
for (var i = 0; i <= 5; i++) {
 (function () {
 	var j = i;
 	setTimeout(function timer() {
 	    console.log(j);
 	}, i*1000);
 })();
}
```

这样就直接实现了我们想要的结果。不过用 js 实现这么简单的功能还要这么复杂，好在现在有 ES6，用 let 来进行声就直接搞定明：

```c
for (let i = 0; i <= 5; i++) {
	setTimeout(function timer() {
		console.log(i);
	}, i*1000);
}
```

以上便是我对 js 的作用域学习做的一些总结，把上述的例子弄懂应该就能很好理解这一概念。

参考：[You Don't Know JS: Scope & Closures](https://github.com/getify/You-Dont-Know-JS/blob/master/scope%20&%20closures/README.md#you-dont-know-js-scope--closures)







