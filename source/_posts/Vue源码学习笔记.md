---
title: Vue源码学习笔记
date: 2017-12-01
---

> 最近偷懒好久没有写博客了，一直想继续Vue学习系列，想深入Vue源码来写。结果发现自己层次不够，对js的理解差好多。所以一直想写一直搁置着。最近重新振作决心看完Vue源码，并且以我们这类前端小白的角度来一步步弄懂Vue源码。

**PS：**以下文章为笔记类，记录了本人在看源码过程中的一些问题和感悟。

# Vue源码的本质是什么
Vue.js 本质上就是一个包含各种逻辑的一个function。而我们通常初始化Vue的过程就是实例化的过程。
```
var vm = new Vue({})
```
话不多说，老规矩用代码说话！
让我们来对Vue进行打印：
```
var vm = new Vue({
  ...
})

console.log(Vue)
console.log(vm)
```
打印结果如图：
![vue](http://upload-images.jianshu.io/upload_images/1987062-51c2e9112e90eeef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里可以看到Vue$3这个方法，就是这个方法对Vue对象的构造函数了。其实这很简单，我们自己都可以造出一个Vue$4
```
    function Vue$4(options) {
        this.options = options
    }

    Vue$4.prototype.name = "小东西"
    Vue$4.prototype.age = 27

    console.log(new Vue$4("很好"))
```
显示结果如图
![vue$4](http://upload-images.jianshu.io/upload_images/1987062-ff4ba64d71f9d5b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

综上所述，Vue对象的本质就是一个function，与我们的Vue$4的不同之处只在于逻辑的多与少。

# 必须理解Object对象
在Vue的源码中，出场率最多的应该就数`Object`对象的使用上了。可以这么说，不懂Object都没法往下看代码。以下是源码中用到的比较多的。

* [Object.defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 直接在一个对象上定义一个新属性，或者修改一个对象的现有属性， 并返回这个对象。
* [Object.create](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/create) 使用指定的原型对象及其属性去创建一个新的对象。
* [Object.keys](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/keys) 返回一个由一个给定对象的自身可枚举属性组成的数组，数组中属性名的排列顺序和使用 for...in 循环遍历该对象时返回的顺序一致 （两者的主要区别是 一个 for-in 循环还会枚举其原型链上的属性）。
* [Object.prototype](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/prototype) Object的原型对象。
* [Object.freeze](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) 冻结一个对象，冻结指的是不能向这个对象添加新的属性，不能修改其已有属性的值，不能删除已有属性，以及不能修改该对象已有属性的可枚举性、可配置性、可写性。也就是说，这个对象永远是不可变的。该方法返回被冻结的对象。

那么，Vue中哪个是Object呢？我们继续试验：
```
console.log(typeof Vue)
console.log(typeof new Vue())
```
输出结果
```
function
object
```
结果显示，使用new来创建的Vue实例就是个对象，所以一切对Object的操作行为都是针对Vue实例对象的。

# 理解setter和getter
在网上看Vue的评论是经常会听到说

> Vue无非就是setter和getter方法的运用而已

这让我等新手一脸懵逼，这里我们就来认识认识setter和getter。
当我们在获取一个Vue实例data中的某个对象，如果你用console打印出来会发现，对象属性中除了常规的对象属性和__proto__对象之外还会多一个set和一个get方法，所谓的setter和getter就是它们。Vue给每一个对象属性都添加了Observer观察数据的获取和修改。好吧，贴代码一睹真容。
```
function defineReactive (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  var dep = new Dep();

  // Object.getOwnPropertyDescriptor() 方法返回指定对象上一个自有属性对应的属性描述符。
  //（自有属性指的是直接赋予该对象的属性，不需要从原型链上进行查找的属性）
  // 对象、属性名称、描述~
  var property = Object.getOwnPropertyDescriptor(obj, key);
  // 当且仅当该属性的 configurable 为 true 时，该属性描述符才能够被改变，同时该属性也能从对应的对象上被删除。默认为 false。
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  var getter = property && property.get;
  var setter = property && property.set;

  var childOb = !shallow && observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      if (Dep.target) { // Watcher
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
          if (Array.isArray(value)) {
            dependArray(value);
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if ("development" !== 'production' && customSetter) {
        customSetter(); // 自定义setter
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = !shallow && observe(newVal);
      dep.notify();
    }
  });
}
```
代码太长？懵逼了？没关系，我们自己来造一个简单的setter和getter来了解一下。
其实用的就是Object.defineProperty方法中就有set和get。

* get 一个给属性提供 getter 的方法，如果没有 getter 则为 undefined。该方法返回值被用作属性值。默认为 undefined。
* set 一个给属性提供 setter 的方法，如果没有 setter 则为 undefined。该方法将接受唯一参数，并将该参数的新值分配给该属性。默认为 undefined。

好了，写代码~
```
    var obj = {}
    var mValue = "abc"
    Object.defineProperty(obj, "_name", {
        configurable: false, 
        enumerable: false, 
        get: function reactiveGetter () {
            return mValue
        },
        set: function reactiveSetter (val) {
            mValue = val
        }
    })

    obj._name = "rose"

    console.log(obj)
```
到这里我们打印log，对象属性的set和get方法就出现了。
![打印结果](http://upload-images.jianshu.io/upload_images/1987062-c11e06d42cf94a2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 逻辑运算符的使用
在源码中有很多[逻辑运算符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Logical_Operators)的使用，有些运用的很巧妙。这里也科普下吧~

首先知道下可以转换成false的值，如下：
* null
* NaN
* 0
* 空字符串（""）
* undefined

### 用法一：判断条件返回true或者false
这是最基本的用法。
```
if (a & a.master & a.master.name) {} // 如果这三个属性都为true值，执行if逻辑
if (a || b) {} // 如果a或者b为true值，执行if逻辑。
```
### 用法二：判断并返回条件对象

* `&&` 如果几个条件都为true，则返回最后一个条件。
* `||` 几个条件从前往后逐一判断，如果那个条件为true，返回该条件，否则返回最后一个条件。
```
var getter = property && property.get;  // 如果两个属性都存在，将property.get赋值给getter
e && e.__ob__ && e.__ob__.dep.depend(); // 如果三个属性都存在，执行第三条语句的方法
var res = assets[id] || assets[camelizedId] || assets[PascalCaseId]; 
// 给res赋值，如果assets[id]为true，则将其传res；
// 如果assets[camelizedId]为true，将其传给res；
// 如果前两者都为false，将assets[PascalCaseId]传给res。
var strat = strats[key] || defaultStrat;
```
然后，我对各种情况做了试验得出以下结论：

* `&&` 判断中，判断值都为 `true`，返回最后一个判断值；判断值中有 `false`
 值，返回第一个 `false` 值。
* `||` 判断中，判断值都为 `true`，返回第一个判断值；判断值中有 `true` 值也有 `false` 值，返回第一个为 `true` 的判断值；如果判断值都为 `false`，返回最后面的 `false` 值。

**注意：**这里所说的返回 `false` 值不一定是 Boolean 类型的 `false`，也可能是 `0、null` 等非值，详见上文。

### 用法三：使用两个非
两个感叹号会确保参数为非值时只能为false，不会是0、空字符串、undefined等非值。
```
if (options) {
    this.deep = !!options.deep;
    this.user = !!options.user;
    this.lazy = !!options.lazy;
    this.sync = !!options.sync;
  } else {
    this.deep = this.user = this.lazy = this.sync = false;
  }
```

# 数组操作
### 复制
先看一段代码。
```
  var a = [ 'jack', 'rose', 'wade' ]
  var b = a
  b[1] = 'marry'
  console.log(a)
  console.log(b)
```
最后的结果是a和b的数组第二个值都变成了marry，原因就是b并不是获得了数组内容，而只是指向了a，a和b其实是一回事。如果我们需要复制，可以这么写：
```
  var b = a.slice()
```
查阅[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/slice)可知：
 > slice() 方法返回一个从开始到结束（不包括结束）选择的数组的一部分浅拷贝到一个新数组对象，原始数组不会被修改。
### 清空
想要将非空数组的内容清空，最快捷的方式如下：
```
arr.length = 0
```
### 合并
直接引用[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/concat)的例子。
> concat() 方法用于合并两个或多个数组。此方法不会更改现有数组，而是返回一个新数组。
```
var arr1 = ['a', 'b', 'c'];
var arr2 = ['d', 'e', 'f'];

var arr3 = arr1.concat(arr2);

// arr3 is a new array [ "a", "b", "c", "d", "e", "f" ]
```
### 遍历
呃，使用forEach方式来遍历数组是我刚知道的事 - -（之前都是用的for方法……），顺便记录下。
```
arr.forEach(function(obj){
    // 数组循环
})
```
# 几种特殊的写法
在代码中有些代码片段有些看不懂，于是在SF上进行了提问：[看Vue源码，有两段代码写法不知是何意思，求指教~](https://segmentfault.com/q/1010000012573448)。感谢大家的帮助，这里总结下。
第一段是一段在{}里的代码。
```
{
    dataDef.set = function (newData) {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      );
    };
    propsDef.set = function () {
      warn("$props is readonly.", this);
    };
}
```
这段代码，有的朋友说是块级作用域、隔离作用域。不过另一种说法更可信。那就是我所看到的vue是编译完的的代码。源代码其实在大括号前是有条件的。
```
if (process.env.NODE_ENV !== 'production') {
  ...
```
再来看下面的代码
```
(function (global, factory) {
    typeof exports === 'object' && typeof module !== 'undefined' ? module.exports = factory() :
    typeof define === 'function' && define.amd ? define(factory) :
    (global.Vue = factory());
}(this, (function () { 'use strict';

})))
```
这段代码是Vue开头的一段代码，它有两个知识点。
* 立即执行函数 —— 定义函数并立即执行，写法有 `(function(){})()` 或者 `(function(){}())` 的形式。
* 由于过去前端没有模块系统，使用script标签引入的js脚本共享同一个作用域，如果不把代码包起来，很容易产生作用域污染、变量冲突的问题。

**PS：**这两个问题真是网友的结论，不一定完善。有空我会查资料证实~

# 其他知识点
*  [Arguments](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/arguments) —— 传给函数的参数数组
* [new运算符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/new) —— 通过构造函数创建对象
* `"development" !== 'production’`的作用 —— webpack打包判断执行环境是不是生产环境，如果是生产环境会压缩并且没有提示警告之类的东西
* [ instanceof](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/instanceof) —— 验证实例对象是否为该构造函数new出来的。
* [in关键字](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/in) —— 判断某个值是否在数组或对象中。
* [Proxy对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) —— 创建某个对象，并定义一些行为给该对象。
* 字符串的 `charAt` 是获取第几个字符，而 `slice` 方法是截取某段字符。
* [delete关键字](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/delete) —— 用于删除对象中的某个关键字。
* [call()方法](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/call) —— 函数的调用，第一个参数为this，之后为函数定义参数。试验了下 `fun(a, b)` 和 `fun.call(this, a, b)` 两种写法效果一致。另外还可以看下 [apply()方法](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)

**PS：**都是从MDN中找到的资料，多查MDN让我受益良多。



# 未完待续
还在学习Vue源码中……看了很多文章、也看了一遍源码。内容太多，千头万绪，容我理清之后，用自己的文字把Vue的源码学习记录分享出来。一些值得记录下来的知识点和心得会继续在本文中更新。
最后，想学习Vue源码的同学可以去买[《Vue.js权威指南》](https://item.jd.com/12028224.html?dist=jd)这本书，虽然许多章节内容和[官网](https://cn.vuejs.org/v2/guide/)是重复的，不过源码解析部分值得一看。我也正配合着这本书和源码在学习Vue。


