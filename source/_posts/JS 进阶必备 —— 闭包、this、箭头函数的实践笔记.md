---
title: JS 进阶必备 —— 闭包、this、箭头函数的实践笔记
date: 2018-04-01
---

> 闭包、this 和箭头函数是三个常见面试题，也是 js 进阶之路上的拦路虎。这次还用实践熟悉这三个问题。

# this 实践
## demo
写了一段代码来验证 this 的指向。
```js
    console.log('全局环境的this', this)
    
    function test (){
        console.log('方法中的 this', this)

        function child(){
            console.log('方法中的方法的 this', this)
        }
        child()
    }
    test()

    var obj =  {
        name: "object",
        doSth: function() {
            var oName = "obj function name"
            console.log('对象中的 this', this)
        },
        childObj: {
            name: "childObj",
            doSth: function() {
                var arrow = () => {
                    console.log("对象中箭头函数的this", this)
                }
                arrow()
                console.log('对象的对象中的 this', this)
            }
        },
        doBibao: function() {
            var count = 500
            console.log('闭包构造方法的 this', this)
            return function() {
                console.log('闭包返回结果的 this', this)
            }
        }
    }
    obj.doSth()
    obj.childObj.doSth()
    var mbb = obj.doBibao()
    mbb()

    setTimeout(obj.doSth, 1800)
    setTimeout(obj.doSth.bind(obj), 2000)

    var fun = function() {
        console.log('匿名函数中的 this', this)
    }
    fun()

    class Vue {
        constructor(options){
            this.name = "vue"
            this.type = "object"
            this.options = options
            console.log('构造函数的 this', this)

            options.log()
        }
    }

    var vm = new Vue({
        log: function () {
            console.log('构造函数找那个传递方法的 this', this)
        }
    })

    function bibao (){
        var count = 101
        console.log('闭包外的 this', this)
        return function() {
            count++
            console.log('闭包中的 this', this)
            return count;
        }
    }
    var bi = bibao()
    console.log(bi())

    var mArrow = () => {
        console.log('箭头函数中的 this', this)
    }
    mArrow()
    console.log('以下内容为异步执行')

    setTimeout(() => {
        console.log('延时箭头函数中的 this', this)
    }, 1000)
```

最后结果如图：
![console.log](https://upload-images.jianshu.io/upload_images/1987062-4c6444e7445a41e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



从中可以得到一些结论（以下都是非严格模式下的测试结果）：

* 全局变量的 this 指向 Window。
* 全局变量中的具名函数、匿名函数、闭包函数、箭头函数都指向 Window。
* 在对象中同步调用，this 指向当前对象。
* 在对象中异步调用，this 已重新指向 Window，如果需要指向对象需要使用 `bind()` 方法改变 this 指向。

个人理解：this 作为上下文始终会有一个唯一指向对象。这个对象要么指向 Window 要么指向当前对象。**当调用对象中方法时，方法中的 this 即指向方法。之后 this 会重新指向 Window。**

## call & apply & bind

说到 this 不得不说下函数的间接调用方法 apply 和 call 了。他们作用相同，唯一不同点在于 apply 方法的第二个参数接收一个参数数组。而 call 方法接收若干个参数。这两个方法的第一个参数传递的都是 this 指向。
```js
    function sayHello(p1, p2) {
        console.log(`hello ${p1} and ${p2}`)
    }

    sayHello('jack', 'rose')
    sayHello.call(this, 'jack', 'rose')
    sayHello.apply(this, [ 'jack', 'rose' ])
```
如上代码，其实输出结果是一样的。第一种写法是一种语法糖，第二种和第三种才是真正的方法执行。可以看到它们的第一个参数为 this。**所以，call 方法和 apply 方法是可以改变函数的 this 指向的。**
而我们之前提到的 bind 函数用于重新指定函数的 this 并且创建出一个新的函数的。引用 MDN 上的说法就是：
> bind() 方法创建一个新的函数, 当被调用时，将其this关键字设置为提供的值，在调用新函数时，在任何提供之前提供一个给定的参数序列。

# 箭头函数和一般函数的区别

箭头函数使用更加简洁的表达方式替代匿名函数而深受喜爱。然而箭头函数与一般匿名函数的不同点在于：**箭头函数拥有静态的上下文环境，不会因为不同的调用而改变。**
以下是我理解的静态函数与一般函数的不同点：

* 在箭头函数中，this 是静态的不可变的，所以bind、call和apply方法修改this不起作用：
```js
    var mor = {
        name: "mor"
    }

    var ar = () => {
        console.log("箭头函数的this变化", this)
    }

    ar() // Window
    ar.call(mor, 123) // Window 如果是匿名函数则返回 mor 对象结果
    var newAr = ar.bind(mor)
    newAr() // Window 如果是匿名函数则返回 mor 对象结果
```
* 箭头函数拥有静态的上下文环境，不会因为不同的调用而改变。如下例子中箭头函数的 this 指向了 Window 对象。
```js
    var person = {
        sex: "male",
        age: 28
    }

    person.log = function(){
        console.log("01:" + this.sex + "-" + this.age) // 指向 person 对象
    }

    person.log02 = () => {
        console.log("02:" + this.sex + "-" + this.age) // 指向 Window
    }

    person.log() // 01:male-28
    person.log02() // 02:undefined-undefined
```

# 闭包函数的原理和用途

这里来简单实现一个闭包：
```js
    function add() {
        var a = 100
        var b = 50
        return function(){
            console.log(a + b)
        }
    }

    var count = add()
    count() // 150
```
我对闭包的理解就是：**在函数中返回函数表达式的写法。**

闭包的主要用途有：
* 避免被垃圾回收机制回收方法结果和变量，使变量始终保存在内存中，实现缓存的功能。如需清空缓存需要将值变为 null。
* 通过闭包获取方法中的局部变量。

# 最后

注意，以上都是本人对于这些知识点的理解，可能会有描述不太准确的地方。如有错误还请评论指出，万分感谢。
这里简单记录了一下我对于 this、闭包和箭头函数的理解~更多内容可以看下我提供的参考资料。

# 参考资料

* https://github.com/zchen9/code/issues/1
* https://segmentfault.com/a/1190000006875662
* https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/Arrow_functions
* http://www.ruanyifeng.com/blog/2009/08/learning_javascript_closures.html
* https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Closures