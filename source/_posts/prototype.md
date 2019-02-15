---
title: 聊聊 JavaScript 的原型
date: 2019-02-12
tag: "JavaScript 基础"
---

> JavaScript 中的原型也是一个非常让人头疼的东西，很多前端同学对此也是一知半解，比如我。今天我们就好好捋一捋这个原型。

# 创建对象的方式

下面就是创建对象的几种方式：

```js
var o1 = {
    a: 123,
    b: 'hello world'
}
console.log(o1.b)


function fun2() {
    this.a = 33
    this.b = 'hello o2'
}

var o2 = new fun2()
console.log(o2.b)

class Fun3 {
    constructor() {
        this.a = 365
        this.b = 'hello class'
    }
}
var o3 = new Fun3()
console.log(o3.b)
```

有人说这是三种创建方式，但是我认为其实是两种创建方式（因为 class 语法糖的本质还是 function）：**直接定义对象**和**使用 new 关键词构造对象**。

# 原型和原型链

当我们创建了一个对象之后，就产生了原型（`Object.create(null)` 是特例）。

## prototype 和 `__proto__` 的区别

`__proto__` 是一个非正式的属性，很多环境中不支持该属性。它指向当前对象的原型。如下图：

![__proto__](https://upload-images.jianshu.io/upload_images/1987062-1002cf3d69276266.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的代码是一段原型继承，可以看到对象 obj1 继承了对象 obj，所以 obj1 的 `__proto__` 就指向了 obj，而 obj 的 `__proto__` 则指向了 Object。所有对象的原型链最终都将指向 Object。

而关于 prototype 我摘录了一段话：

> 当你创建函数时，JS 会为这个函数自动添加 `prototype` 属性，值是一个有 constructor 属性的对象。而一旦你把这个函数当作构造函数（`constructor`）调用（即通过`new`关键字调用），那么 JS 就会帮你创建该构造函数的实例，实例继承构造函数 `prototype` 的所有属性和方法。

![prototype](https://upload-images.jianshu.io/upload_images/1987062-c600260dc91c2314.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，对象 bar 的 `__proto__` 属性指向了函数 func 的 `prototype`。

总结下，`__proto__` 指向原型，而 `prototype` 是函数独有且构造的对象原型指向 `prototype`。

## 理解原型链

每个对象都是原型，而对象之间是可以继承的。所以就产生了原型链。看图说话：

![原型链](https://upload-images.jianshu.io/upload_images/1987062-aa7441f2c85bb6f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很好理解了，我们创建了四个对象逐层进行原型继承。最后打印 obj3 对象可以看到 `obj3 -> obj2 -> obj1 -> obj -> Object` 这就是原型链。

如果我要在 obj3 对象上访问 a 属性，那么 JavaScript 就会顺着原型链逐层往下找，最终在 obj 对象上找到了a 属性，这就是原型链查找数据的方式。如果找到 Object 也没有找到属性就返回 `undefined`。

![原型链查找](https://upload-images.jianshu.io/upload_images/1987062-07ac0815c99f5d7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 为对象指定原型的两种方式

那么如何为对象添加原型呢？

## 1. new 关键字

第一种就是通过构造器的方式来创建。

```js
function Foo () {
    this.a = 11
    this.b = 22
}
Foo.prototype.c = 33
Foo.prototype.func = () => {
    console.log('hello')
}

var f = new Foo()

console.log(f)
console.log(Object.getPrototypeOf(f))
```

当然，不得不说的是 ES6 的 class 语法糖写法：

```js
class Foo {
    constructor() {
        this.a = 11
        this.b = 22
    }

    func() {
        console.log('hello')
    }
}

var f = new Foo()
```

两者其实是一样的效果，但是 class 写法更接近常规的类写法。（终于可以让 function 回归它原本的作用上了。）

## 2. Object.create(obj) 面向对象

Object.create() 可以很好的实现原型继承行为，也能通过 Object API 来修改原型：

```js
var obj = { a: 123, b: 456 }
Object.setPrototypeOf(obj, { c: 789 })
var obj2 = Object.create(obj)
obj2.e = 555
```

代码输出结果如下图，的确实现了为对象指定原型的行为。

![原型继承和修改](https://upload-images.jianshu.io/upload_images/1987062-b592d69f559248af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 引用流还是复制流？

使用 JavaScript 原型是特别要主义的一个点是：**JavaScript 对于原型的继承是一种引用行为**，即所引用的对象改变，继承对象的原型也会改变。

与之相反的，有些语言会使用复制的方式。即在原型继承时复制一份原型到当前对象，从此被复制的对象和复制对象再无瓜葛。

# 总结

随着 Object.create() 等一系列新 API 和 ES6 的 class 写法的出现，使用 function 作为构造器并使用 prototype 来修改原型的方式将逐渐被抛弃。但是由于历史原因这部分知识还是要理解其中原理的。

而 `__proto__` 属性是非正式属性，不适合在通用场景下使用。

而对于原型的写法，我认为有两种不错的处理方式：

> 1. 完全使用 class 构造器写法来替代使用 function 构造器的写法来进行**面向类**的开发方式。
> 2. 放弃原型写法，使用 Object 系列 API 进行**面向对象**的开发（行为委托就是这样的方式）。

# 最后

关于原型，先聊这么多。明天我们聊聊基于 Object API 来实现的面向对象模式 —— 行为委托，敬请期待。
