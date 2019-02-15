---
title: JavaScript 类型全知道
date: 2019-02-14
tag: "JavaScript 基础"
---

> 今天来聊聊 JavaScript 的类型。

# JavaScript 的七大基本类型

* undefined 未定义
* null 空值
* boolean 布尔值
* string 字符串
* number 数字
* object 对象
* symbol 符号（ES6）

怎么知道是有这么七个值呢，使用 typeof 运算符来查看。

![typeof](https://upload-images.jianshu.io/upload_images/1987062-43607c1c53f7dca8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中，大多数都如我们想的那样，但是有两个特例：

1. 可以看到 `typeof null` 理论上应该返回 `'null'` 但却返回的是 `'object'`，这是一个存在20多年由来已久的bug，所以要判断对象是否为 null 时需要注意。
2. 当我们打印 `typeof function() {}` 的时候返回的类型是 `'function'`，是不是说明 function 也是基本类型呢？但其实 function 是 object 的子集，下面说引用类型的时候会提到。

# null、undefined 和 undeclared

在 JavaScript 的类型中有三种表示变量“不存在”的方式，null、undefined 和 undeclared。那么它们的区别是什么呢？看代码~

![三种空](https://upload-images.jianshu.io/upload_images/1987062-354f1c86c7c9d36e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 我们定义了变量 a 却没有给他赋值，所以 a 就是 undefined
* 我们没有定义变量 b，所以报错 b is not defined，我们称之为 undeclared。注意这和 undefined 是有区别的。
* 我们定义了变量 c 并给他赋值 null，所以 b 就是 null

小结下：undefined 是定义了变量却没有赋值；undeclared 是没有定义变量更没有赋值，会报错；null 是定义了变量并且赋值空值 null。

# JavaScript 中的那些引用类型

所有的引用类型都是 Object。我猜测由于在 JavaScript 中对于 Object 的访问是引用形式的，所以称之为引用类型。

## Object

Object 是一个函数，它可以用于创建对象，也可以用它带的 API 方法操作对象。

```js
// 创建对象
var obj = new Object(); // 不推荐
var obj2 = {}

// Object API
var obj3 = Object.create(null)
var object2 = Object.freeze(object1);
```

为什么可以使用这些 API？因为在 Object 函数的原型（关于原型可以看之前的文章）中有定义这些 API 方法。

![api 方法在这里](https://upload-images.jianshu.io/upload_images/1987062-2ecc22f6b5162ed1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其他更多的 Object API 可以查阅 [Object - JavaScript | MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object)。

## Array

同样的，Array 函数用于创建数组和提供操作数组的 API。

```js
// create array
var arr1 = []
var arr2 = new Array()
var arr3 = Array()

// api
Array.isArray(arr1);
arr1.push(1)
```

同样的，推荐查阅 [Array - JavaScript | MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array) 获取更多 API 信息。

## Date

Data 用于创建时间对象。

```js
var now = new Date()
```

Date 函数通过传递不同的参数在生产不同的时间对象，参考 [Date - JavaScript | MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Date)。

注：在 ES6 中可以通过静态方法 Date.now() 来获取当前时间。

## RegExp

RegExp 构造函数创建了一个正则表达式对象，用于将文本与一个模式匹配。

```js
var regex1 = /\w+/; 
var regex2 = new RegExp('\\w+'); // 不推荐
```

## Function

Function 构造函数 创建一个新的Function对象。 在 JavaScript 中, 每个函数实际上都是一个 Function 对象。

```js
var e = new Function( "a", "return a * 2;" ); // 不推荐
var f = function(a) { return a * 2; }
function g(a) { return a * 2; }
```

## 包装器对象（Boolean、String 和 Number）

> Boolean、String 和 Number 分别是基本类型 boolean、string 和 number 的包装器对象，有很多共性，所以就拿来一起讲了。

### 创建值

```js
var bool = true
var bool2 = Boolean(true) // 不推荐

var str = 'hello world'
var str2 = String('hello world') // 不推荐

var num = 100
var num2 = Number(100) // 不推荐
```

可以看到，第二种创建方式非常的画蛇添足，但是这种写法可以有别的用处。

### 自动包装

上面我们定义了三个不同基本类型的变量，这几个变量后面可以加一些方法来进行操作。

```js
bool.toString();
str.split(' ')
num..toFixed(2)
```

为什么明明基本类型没有这些属性和方法却可以使用呢？
这就要提到这三个基本类型的自动包装特性了。即虽然这三个基本类型没有属性，但是当我们调用其属性和函数时，会自动包装成相应的对象。

```js
new Boolean(bool).toString()
new String(str).split(' ')
new Number(num).toFixed(2)
```

可以看下 String 的原型中的确包含了 string 类型用到的所有的属性和方法。

![string](https://upload-images.jianshu.io/upload_images/1987062-1e612c6730b31699.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 强制类型转换

对象包装器另外一个作用 —— 强制类型转换：

```js
Boolean(''); // false
Boolean(1); // true

String(null); // 'null'
String(12138); // '12138'

Number("123")  // 123
Number("") // 0
```

直接使用包装器函数就可以对值进行强制类型转换行为。

## Global

Global 表示全局对象，具体的内容暂时没有找到，先贴一段网上的解释：

> 《JavaScript高级程序设计》中谈到，global对象可以说是ECMAScript中最特别的一个对象了，因为不管你从什么角度上看，这个对象都是不存在的。从某种意义上讲，它是一个终极的“兜底儿对象”，换句话说呢，就是不属于任何其他对象的属性和方法，最终都是它的属性和方法。我理解为，这个global对象呢，就是整个JS的“老祖宗”，找不到归属的那些“子子孙孙”都可以到它这里来认祖归宗。所有在全局作用域中定义的属性和函数，都是global对象的属性和方法，比如isNaN()、parseInt()以及parseFloat()等，实际都是它的方法；还有就是常见的一些特殊值，如：NaN、undefined等都是它的属性，以及一些构造函数Object、Array等也都是它的方法。总之，记住一点：global对象就是“老祖宗”，所有找不到归属的就都是它的。

## Math

Math 是一个内置对象， 它具有数学常数和函数的属性和方法。不是一个函数对象。举几个栗子：

```js
Math.abs(-1);     // 取绝对值 1
Math.max(10, 20);   //  取最大值 20
Math.random(); // 取随机数
```

## Symbol

Symbol 函数是 ES6 添加的，用于表示有唯一性的特殊值。创建方法如下：

```js
var a = Symbol('this is a symbol')
```

## Error

Error 函数用于创建错误对象。

```js
var err1 = new Error('I was created using a function call!')
var err2 = Error('I was created using a function call!')
```

# 最后

本文先是介绍了七大基本类型，之后有简单介绍了 JavaScript 中的引用类型。主要想达到归纳整理的作用，让大家知道都有哪些类型，而具体使用中则强烈推荐查阅 [MDN](https://developer.mozilla.org/zh-CN/) 来使用各种方法和属性。

希望本文对你有所帮助~明天聊聊我最近从 Vue 转 React 项目的一些体会，明天见！