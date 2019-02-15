---
title: 聊聊 JavaScript 的 this
date: 2019-02-11
tag: "JavaScript 基础"
---

JavaScript 中的 this 一直是比较让人头疼，也是面试特别容易问及的问题。下面就参照这《你不知道的 JavaScript》来学习下 this 这个神奇的东西。

# this 到底指向何处

> this 是在运行时进行绑定的，并不是在编写时绑定，它的上下文取决于函数调用时的各种条件。 this 的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。

所以，this 并不只是简单地指向函数或者对象自身。

# this 的四种绑定方式

## 默认绑定

所谓的默认绑定就是 this 的默认绑定方式。

```js
function foo() {
    console.log(this.a)
}

var a = 3;

foo(); // 3
```

**注意：**严格模式下这种默认绑定形式不成立。

```js
var a = 3;

function foo() {
    "use strict";
    console.log(this.a)
}

foo(); // TypeError
```

## 隐式绑定

隐式绑定是指 this 所在函数在有**上下文**的前提下的绑定，如 `obj.foo();`。

```js
function foo() {
    console.log(this.a);
}

var a = 1;
var obj = {
    a: 2,
    foo: foo
};

obj.foo(); // 2
```

**注意：**对象中的函数只是引用关系，即对象和函数存在于两个地方。所以在别的地方使用函数，与隐式绑定的对象就没有关系了。看下两个例子：

1. 其中 `var myFoo = obj.foo` 的 myFoo 变量引用的是 foo() 函数，与 obj 并无关系。所以 myFoo 的执行函数行为就变成了默认绑定，打印结果为 1。

```js
function foo() {
    console.log(this.a);
}

var a = 1;
var obj = {
    a: 2,
    foo: foo
};

var myFoo = obj.foo
myFoo(); // 默认绑定，值为 1
```

2. 在回调函数中其实也会出现 this 绑定丢失的情况，回调函数 obj.foo 引用的是 foo 函数，与 obj 对象并无关系。

```js
function foo() {
    console.log(this.a);
}

var a = 1;
var obj = {
    a: 2,
    foo: foo
};

setTimeout(obj.foo, 300); // 1
```

## 显示绑定

显式绑定就是指使用 call、apply、bind 来指定某个上下文进行绑定，它们的一个作用就只为函数硬绑定一个上下文对象。

之前的回调函数使用 bind 进行修改后打印出了我们 obj 对象中的 a 属性：

```js
function foo() {
    console.log(this.a);
}

var a = 1;
var obj = {
    a: 2,
    foo: foo
};

setTimeout(obj.foo.bind(obj), 300);
```

call 和 apply 也是类似的，通过对函数指定上下文来进行硬绑定，且硬绑定只能绑定一次。

> `call()` 方法的作用和 `apply()` 方法类似，区别就是 `call()` 方法接受的是参数列表，而 `apply()` 方法接受的是一个参数数组。

## new绑定

new 关键字创建对象的过程其实也是一个绑定上下文的过程，所以使用 new 创建的对象的 this 也要格外注意。

> 使用 new 来调用函数，或者说发生构造函数调用时，会自动执行下面的操作。
>
> 1. 创建（或者说构造）一个全新的对象。
> 2. 这个新对象会被执行 [[ 原型 ]] 连接。
> 3. 这个新对象会绑定到函数调用的 this 。
> 4. 如果函数没有返回其他对象，那么 new 表达式中的函数调用会自动返回这个新对象。

```js
function foo(a) {
  this.a = a;
}
var bar = new foo(2);
console.log( bar.a ); // 2
```

可以看到 new 行为的第三步就是进行 this 绑定，我们也可以从代码看到 new 行为的确有绑定 this 的能力。

## this 四种绑定方式排序

既然四种绑定都能够改变 this 的指向，那么这四种绑定的优先级是怎样的呢？结论是：

```
new 绑定 > 显示绑定 > 隐式绑定 > 默认绑定
```

虽然很少会出现多个场景绑定一个 this 的情况，但是知道下也能以防万一。

# 箭头函数

关于 this 最后要说的就是 ES6 的箭头函数。

```js
function foo() {
    setTimeout(() => {
        // 这里的 this 在此法上继承自 foo()
        console.log(this.a);
    }, 100);
}
var obj = {
    a: 2
};
foo.call(obj); // 2
```

它完全等同于：

```js
function foo() {
    var self = this; // lexical capture of this
    setTimeout(function () {
        console.log(self.a);
    }, 100);
}
var obj = {
    a: 2
};
foo.call(obj); // 2
```

关于箭头函数只要记住 `var self = this;` 就够了。

它其实是通过词法作用域保存当前 this 上下文传递给回调函数。本质上是抛弃了 this 原有的机制。

# 最后

我们从四种常见 this 绑定方式和箭头函数这两个角度系统的学习了 this 绑定的知识点，相信之后你再也不怕 this 相关的知识点了！

本文还有很多可以改进的地方，如有任何意见和问题，欢迎留言指出。谢谢~

# 推荐资料

* 你不知道的 JavaScript （上册）
* [Know this, use this! (总结 this 的常见用法)](https://juejin.im/entry/57c25064d342d3006b216070)