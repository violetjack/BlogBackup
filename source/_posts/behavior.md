---
title: 聊聊对象行为委托
date: 2019-02-13
tag: "JavaScript 基础"
---

> 昨天我们聊到了对象的原型，知道了有 new 和 Object.create() 两种操作原型的方式。今天我们来对比下使用这两种方式进行面向对象编程的特点。

# 使用 new 关键字写面向类

先来一段面向类的代码实现

```js
function Foo(who) {
    this.me = who;
}
Foo.prototype.identify = function () {
    return "I am " + this.me;
};

function Bar(who) {
    Foo.call(this, who);
}
Bar.prototype = Object.create(Foo.prototype);
Bar.prototype.speak = function () {
    console.log("Hello, " + this.identify() + ".");
};

var b1 = new Bar("b1");
var b2 = new Bar("b2");

b1.speak();
b2.speak();
```

当然也可以用 ES6 的 class 语法糖。class 的出现避免了在函数的 `prototype` 上添加属性的奇怪的行为。

# 混乱不堪的原型关系

无论是 function 还是 class，其背后还是逃不开对于 prototype 的操作。而原型中各种关系令人头疼。下面是这段代码的关系图：

![原型关系](https://upload-images.jianshu.io/upload_images/1987062-b78fe4bbd7d6532a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

抛开函数与 Function 和 Object 的关系，我们来捋一捋代码中的逻辑。

1. Foo 函数的 prototype 上创建了函数 identify
2. Foo 函数的 prototype 引用了 Object 的原型（原型链理论）
3. Bar 函数 prototype 引用了以 Foo 的 prototype 为原型创建的对象
4. Bar 函数的 prototype 上创建了函数 speak
5. 以 Bar 为构造器分别 new 了两个对象 b1 和 b2，b1 和 b2 的原型引用了 Bar 函数的 prototype
6. 由于 Bar 进行了修改原型的操作，所以没有 constructor 函数。
7. 所以根据原型链理论，Bar 的 prototype 和 b1、b2 的 constructor 都指向了 Foo 函数的 constructor。

就这么个逻辑（这还没有包括函数与对象之前的关系），我表示我讨厌在 prototype 上去添加属性，这显得非常乱。

最后输出的 b1 对象的原型结构如下：

![原型结构](https://upload-images.jianshu.io/upload_images/1987062-1cd13125458907a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 行为委托闪亮登场

可以看到上面的写法导致指向非常混乱，实际上我们并不需要了解这么多逻辑。下面介绍一种方式完全不管 prototype 真正面向对象的写法 —— 行为委托。

就拿上面的例子来说：

```js
Foo = {
    init: function (who) {
        this.me = who;
    },
    identify: function () {
        return "I am " + this.me;
    }
};

Bar = Object.create(Foo);
Bar.speak = function () {
    alert("Hello, " + this.identify() + ".");
};

var b1 = Object.create(Bar);
b1.init("b1");
var b2 = Object.create(Bar);
b2.init("b2");

b1.speak();
b2.speak();
```

其中的关键就是完全使用对象来写面向对象编程，而避免使用 new 一个构造器的写法。这里我们也来理一理逻辑：

1. 创建 Foo 对象，它包含 init 和 identify 两个函数属性。
2. 创建 Bar 对象原型继承 Foo 对象。
3. 在 Bar 对象上添加 speak 方法。
4. 创建 b1 和 b2 对象原型继承 Bar 对象。
5. 使用原型链上的 init 函数为 b1 和 b2 对象传入数据。
6. 通过原型链调用 speak 函数。

这种写法的原型关系图就简单了很多：

![原型关系](https://upload-images.jianshu.io/upload_images/1987062-bdc4c082cbe29864.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到与第一中方法比少了函数构造器，少了函数构造器就没了 constructor 和 prototype。完全通过对象与对象之间的原型继承引用关系来实现面向对象的编程思想。

最后看下输出的 b1 对象的原型结构：

![原型结构](https://upload-images.jianshu.io/upload_images/1987062-20caa15116450797.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 行为委托的好处

* 行为委托避免使用 new 构造器形式来实现面向对象，减少了大量构造器所带出的复杂关系。
* 行为委托只使用对象之间的原型继承关系，让整个代码逻辑变得非常清晰。

# 最后

这里介绍了行为委托这一种面向对象的设计思想，它让面向对象编程变得更加简洁、更加自然。

当然，这只是一种设计方式。如果执意要用 new 写法来写面向对象编程当然没有问题，推荐使用 class 语法糖，它可以将操作 prototype 的行为给隐藏起来，这使得代码更像 Java（引用对象的特性并未改变，所以只是看着像），从而让代码更好理解。

赶快去试试行为委托吧，我认为它是种很适合 JavaScript 的设计模式。

明天我们聊聊 JavaScript 的类型~