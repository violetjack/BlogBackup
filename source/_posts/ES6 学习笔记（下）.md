---
title: ES6 学习笔记（下）
date: 2018-01-21
---

 > 认真学习了一遍ES6，发现很多很好用的功能。
学习资料：[《ECMAScript 6 入门》](http://es6.ruanyifeng.com/#README)

好啦，继续下半部分的学习。

# Proxy
Proxy 用于修改对象的某些操作行为。套用书上的栗子，实现了对象的set、get方法拦截。
```
var obj = new Proxy({}, {
    get: function (target, key, receiver) {
        console.log(`getting ${key}!`);
        return Reflect.get(target, key, receiver);
    },
    set: function (target, key, value, receiver) {
        console.log(`setting ${key}!`);
        return Reflect.set(target, key, value, receiver);
    }
});

obj.count = 1
obj.name = 'jack'
obj.count++

// setting count
// setting name
// getting count
// setting count
```
**实例方法整理**
* get 方法拦截属性的读取操作。
* set 方法拦截赋值操。
* apply 方法拦截函数的调用。
* has 方法拦截 `HasProperty` 操作，即查找对象中是否有某属性。可用来隐藏一些属性不被 `in` 运算符发现。
* construct 方法拦截 `new` 指令。即在 `new` 指令创建实例的时候可以对对象中的参数进行一些初始化修改操作。
* deleteProperty方法拦截 `delete` 指令，可用来保护某些对象属性无法被删除。
* defineProperty方法拦截了Object.defineProperty操作。
* getOwnPropertyDescriptor方法拦截Object.getOwnPropertyDescriptor()，返回一个属性描述对象或者undefined
* getPrototypeOf方法主要用来拦截获取对象原型。
* isExtensible方法拦截Object.isExtensible操作。
* ownKeys方法用来拦截对象自身属性的读取操作。
* preventExtensions方法拦截Object.preventExtensions()。该方法必须返回一个布尔值，否则会被自动转为布尔值。
* setPrototypeOf方法主要用来拦截Object.setPrototypeOf方法。

apply方法的使用
```
var target = function () { return 'I am the target'; };
var handler = {
    apply: function () {
        return 'I am the proxy';
    }
};

var p = new Proxy(target, handler);

console.log(p())
// "I am the proxy"
```
has方法的使用
```
var handler = {
    has (target, key) {
        if (key[0] === '_') {
            return false;
        }
        return key in target;
    }
};
var target = { _prop: 'foo', prop: 'foo' };
var proxy = new Proxy(target, handler);
console.log('_prop' in proxy)
// false
```
construct方法的使用
```
var p = new Proxy(function () {}, {
    construct: function(target, args) {
        console.log('called: ' + args.join(', '));
        return { value: args[0] * 5 + 12 };
    }
});

console.log(new p(1))
console.log(new p(1).value)

// call: 1
// { value: 17 }
// call: 1
// 17
```
所以，我理解的 Proxy 对象主要功能就是拦截对象属性的一些操作。

# Reflect
我理解的Reflect对象：
* 是Object的高级版本，Object对对象的操作方法Reflect对象都有，并且未来操作对象的新方法只放在Reflect对象中有。
* 发生错误不会报错而是返回false，可直接在判断中使用。
* 让Object操作都变成函数行为，统一表现形式。某些Object操作是命令式，比如 `name in obj` 和 `delete obj[name]` ，而 `Reflect.has(obj, name)` 和 `Reflect.deleteProperty(obj, name)` 让它们变成了函数行为。
* Reflect对象的方法与Proxy对象的方法一一对应，只要是Proxy对象的方法，就能在Reflect对象上找到对应的方法。Proxy对象拦截对象属性方法，进行重新定义。而Reflect对象立即执行对象属性方法。下面例子中使用set和get做演示。
```
var myObject = {
    foo: 1,
    bar: 2,
    get baz() {
        return this.foo + this.bar;
    },
}

console.log(Reflect.get(myObject, 'foo')) 
console.log(Reflect.get(myObject, 'bar')) 
console.log(Reflect.get(myObject, 'baz')) 

console.log(Reflect.set(myObject, 'foo', 100))
console.log(myObject.foo)

// 1
// 2
// 3
// true
// 100
```

# Promise 对象
Promise 对象登场啦~这是 ES6 语法中非常常用的对象。
我理解的 Promise 对象：
* Promise 对象让异步操作的写法从回调函数变为链式操作，可读性更强。
* Promise 对象一旦改变，就会锁死，不再改变。

**基本用法**
Promise 的定义，定义一个 Promise对象，参数为resolve和reject，resolve为执行成功的方法，而reject为执行失败的方法。
```
const promise = new Promise(function(resolve, reject) {
  // ... some code

  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});
```
基本调用方式
```
promise.then(function(value) {
  // success
}, function(error) {
  // failure
});
```
**Promise 函数一旦用 new 指令创建，立即执行。并且数据为不可变。**
```
let success = true
let name = 'jack'

const promise = new Promise((resolve, reject) => {
    console.log('create')
    if (success) {
        resolve(name)
    } else {
        reject('error promise')
    }
})

success = false
name = 'rose'
console.log('before then')

promise.then(value => {
    console.log(value)
}).catch(error => {
    console.log(error)
})

// create
// before then
// jack
```
由此可见，在定义了Promise之后，我们再去修改 `name` 返回的还是定义Promise时候的值。说明了Promise对象定义即执行，并且不可变。

**Promise 推荐使用 `promise.then().catch()`  写法。**
```
// bad
promise.then(value => {
    console.log(value)
}, error => {
    console.log(error)
})

// good
promise.then(value => {
    console.log(value)
}).catch(error => {
    console.log(error)
})
```
**then方法链式写法表达，then方法的返回值可以传递给下一个then方法。**
```
const promise = new Promise((resolve, reject) => {
    resolve('jack')
})

promise.then(value => {
    console.log(value)
    return 'violet' + value
}).then(value => {
    console.log(value)
    return 'welcome to ' + value
}).then(value => {
    console.log(value)
    return  value + ' blog'
}).then(value => {
    console.log(value)
})

// jack
// violetjack
// welcome to violetjack
// welcome to violetjack blog
```
**catch方法用于捕获Promise对象的异常行为（可能是 reject 函数返回的错误，也可能是`throw new Error('error')`）。**
**Promise.all() 方法将多个Promise实例包装成一个Promise实例。如下示例，如果p1、p2、p3都执行成功，则执行then方法，返回的参数为三个实例的参数数组；如果有任意一个Promise实例报错，则在catch方法中返回该实例的错误信息。**
```
const p1 = new Promise((resolve, reject) => {
    resolve('jack')
})

const p2 = new Promise((resolve, reject) => {
    // resolve('rose')
    reject('rose error')
})

const p3 = new Promise((resolve, reject) => {
    resolve('james')
})

Promise.all([p1, p2, p3]).then(values => {
    console.log(values)
}).catch(error => {
    console.log(error)
})

// ["jack", "rose", "james"]
// rose error
```
**Promise.race() 方法用法与 all 方法一致，唯一不同点就是多个 Promise实例中只作用域最快有反映的Promise实例，并且返回该实例的正确或错误信息。如果多个Promise同时触发，按顺序返回第一个Promise实例。**
**Promise.resolve() 和 Promise.reject()**
```
Promise.resolve('foo')
// 等价于
new Promise(resolve => resolve('foo'))

const p = Promise.reject('出错了');
// 等同于
const p = new Promise((resolve, reject) => reject('出错了'))
```

# Iterator 和 for...of 循环
**默认Iterator接口部署在数据对象的 `Symbol.iterator` 属性中，`Symbol.iterator` 属性本身是一个函数，就是当前数据结构默认的遍历器生成函数。只要数据对象有了 `Symbol.iterator` 属性就可以进行遍历。如下对象添加了 `Symbol.iterator`  属性后实现了遍历操作。**
```
let obj = {
    data: [ 'hello', 'world' ],
    [Symbol.iterator]() {
        const self = this;
        let index = 0;
        return {
            next() {
                if (index < self.data.length) {
                    return {
                        value: self.data[index++],
                        done: false
                    };
                } else {
                    return { value: undefined, done: true };
                }
            }
        };
    }
};

console.log([...obj])

// ["hello", "world"]
```
**Iterator 接口主要供for...of消费。**
**`for...in` 循环读取键名，`for...of` 循环读取键值。**
```
const arr = ['red', 'green', 'blue'];

for(let v of arr) {
    console.log(v); // red green blue
}
for(let k in arr){
    console.log(k) // 0 1 2
}
```
# Generator 函数的语法
**一种异步解决方案。函数执行返回一个对象，而函数中的数据只有在对象使用 next() 方法才会返回下一个用 `yield` 或者 `return` 定义的数据，否则对象状态就凝固在那里。**
```
function* helloGenerator() {
    yield 'hello'
    yield 'world'
    return 'generator'
}

var h = helloGenerator()
console.log(h.next())
console.log(h.next())
console.log(h.next())
console.log(h.next())

// { value: 'hello', done: false}
// { value: 'world', done: false}
// { value: 'generator', done: true}
// { value: undefined, done: true}
```
**next() 方法传值—— `next()` 方法返回的是 `yield` 表达式的计算结果。如果 `next(value)` 方法中传入value参数，则参数将替换上一个 `yield` 数据。如下示例中，`12` 替换了`(yield (x + 1))`，`13` 替换了`yield (y / 3)`，最后得到结果为42。**
```
function* foo(x) {
  var y = 2 * (yield (x + 1));
  // value  = 5 + 1
  var z = yield (y / 3);
  // y = 2 * 12 value = 24 / 3
  return (x + y + z);
  // z = 13 y = 24 z = 13 value = 5 + 24 + 13
}

var a = foo(5);
a.next() // Object{value:6, done:false}
// 如果不传递数据，则y=NaN
a.next() // Object{value:NaN, done:false}
a.next() // Object{value:NaN, done:true}

var b = foo(5);

b.next() // { value:6, done:false }
b.next(12) // { value:8, done:false }
b.next(13) // { value:42, done:true }
```
**throw方法用于捕捉错误，return方法类似于 Genterator 函数的 `return xxx` 返回某个值，随后再使用next方法返回的都是 `undefined`**
**对于 next、throw、return，引用书上的解释更清晰点。**
> `next()` 是将 `yield` 表达式替换成一个值。
`throw()` 是将 `yield` 表达式替换成一个 `throw` 语句。
`return()` 是将 `yield` 表达式替换成一个 `return` 语句。

`yield*` **用于将其他 Generator 函数合并到当前函数中，用法如下：**
```
function* bar() {
    yield 'a'
    yield 'b'
}

function* foo() {
    yield 'x'
    yield* bar()
    yield 'y'
}

for (let v of foo()){
    console.log(v)
}

// x
// a
// b
// y
```
**Generator 函数不能直接用 new 指令实例化对象，需要包装为普通函数再 new**
```
function* gen() {
  this.a = 1;
  yield this.b = 2;
  yield this.c = 3;
}

function F() {
  return gen.call(gen.prototype);
}

var f = new F();

f.next();  // Object {value: 2, done: false}
f.next();  // Object {value: 3, done: false}
f.next();  // Object {value: undefined, done: true}

f.a // 1
f.b // 2
f.c // 3
```
**自动执行所有Generator函数的方法：**
```
function run(fn) {
  var gen = fn();

  function next(err, data) {
    var result = gen.next(data);
    if (result.done) return;
    result.value(next);
  }

  next();
}

function* g() {
  // ...
}

run(g);
```
**以上自动执行器还可以使用 `co` 模块来实现。**
# async 函数
async函数用于处理异步操作，它是对Generator函数的改进。它相比于Generator有以下几个优点：
* 内置执行器：相比于Generator 要自定义或者用 co 模块来实现自动执行器效果，async函数自带自动执行器。
* 更好的语义：async 和 await，比起 * 和 yield，语义更清楚了。async表示函数里有异步操作，await表示紧跟在后面的表达式需要等待结果。
* 更广的适用性：co模块约定，yield命令后面只能是 Thunk 函数或 Promise 对象，而async函数的await命令后面，可以是 Promise 对象和原始类型的值（数值、字符串和布尔值，但这时等同于同步操作）。
* 返回Promise：async函数的返回值是 Promise 对象，这比 Generator 函数的返回值是 Iterator 对象方便多了。你可以用then方法指定下一步的操作。

我个人对async的感觉是写法方便、代码理解简单、代码写法也符合逻辑、操作异步行为方便。
下面写了 Generator 函数和 async 函数实现异步的代码的对比。
```
const readFile = function (fileName) {
    return new Promise(function (resolve, reject) {
        setTimeout(() => {
            console.log(`reading ${fileName}`)
            resolve(fileName)
        }, 1000)
    });
};

// Generator 写法
const gen = function* () {
    const f1 = yield readFile('/etc/fstab');
    const f2 = yield readFile('/etc/shells');
    console.log(f1.toString());
    console.log(f2.toString());
};

var g = gen()
g.next().value.then(value => {
    g.next(value).value.then(value => {
        g.next(value)
    })
})

// async 写法
async function gan() {
    const f1 = await readFile('/etc/fstab');
    const f2 = await readFile('/etc/shells');
    console.log(f1.toString());
    console.log(f2.toString());
}

gan()
```
两种函数的实现结果是一样的。
但从上面的例子中可以看出，Generator 函数需要不断调用next方法，并且将上一个next方法的结果传递给当前next方法当做参数。而async函数直接调用函数本身就会自动往下执行。Generator多了一步执行的过程。
另外，async await的语义很清晰，就算没学过ES6的大致都能看懂是什么意思啦~

# Class
class其实就是一个函数实例化的语法糖~具体功能也类似Java这类有Class的语言~
所以，下面两种写法的结果是相等的。
**传统写法**
```
function Point(x, y) {
    this.x = x;
    this.y = y;
}

Point.prototype.toString = function () {
    return '(' + this.x + ', ' + this.y + ')';
};

var p = new Point(1, 2);
console.log(p)
```
**ES6 Class写法**
```
class Point {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }

    toString() {
        return '(' + this.x + ', ' + this.y + ')';
    }
}

var p = new Point(1, 2);
console.log(p)
```
![结果](http://upload-images.jianshu.io/upload_images/1987062-3f972d9e15d96308.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对象结果如上图所示，构造函数中this对象的属性在实例中，而Class里面的函数再实例对象的 `__proto__` 中。
**使用 `new target` 在构造函数中判断对象是否为 `new` 指令创建的。**
```
// 另一种写法
function Person(name) {
  if (new.target === Person) {
    this.name = name;
  } else {
    throw new Error('必须使用 new 命令生成实例');
  }
}

var person = new Person('张三'); // 正确
var notAPerson = Person.call(person, '张三');  // 报错
```
# 修饰器
类似于Java的修饰器 `@Override` ， 现有一个提案，将修饰器加入到 ECMAScript 中。

# Module
在 ES6 中添加了模块化功能，很常见用法也很简单。
**模块加载**
```
import { stat, exists, readFile } from 'fs'; // 多个模块加载
import { lastName as surname } from './profile.js'; // 模块加载重命名
import * as circle from './circle'; // 整理加载
```
**模块输出**
```
// 输出变量
export let a = 100
// 输出方法
export function hello() {
    console.log('hello world')
}
// 输出多个变量
const b = 200
const c = 300
const d = 400
export {b, c, d}
// 输出变量重命名
export {b as value}
// 输出默认值
export default function () {
    console.log('hello default')
}
```
**`CommonJS` 语法中模块输出和加载的写法**
```
let { stat, exists, readFile } = require('fs');

module.exports = {
  counter: counter,
  incCounter: incCounter,
};
```
# ArrayBuffer
ArrayBuffer对象、TypedArray视图和DataView视图是 JavaScript 操作二进制数据的一个接口。

> **二进制数组由三类对象组成。**
（1）ArrayBuffer对象：代表内存之中的一段二进制数据，可以通过“视图”进行操作。“视图”部署了数组接口，这意味着，可以用数组的方法操作内存。
（2）TypedArray视图：共包括 9 种类型的视图，比如Uint8Array（无符号 8 位整数）数组视图, Int16Array（16 位整数）数组视图, Float32Array（32 位浮点数）数组视图等等。
（3）DataView视图：可以自定义复合格式的视图，比如第一个字节是 Uint8（无符号 8 位整数）、第二、三个字节是 Int16（16 位整数）、第四个字节开始是 Float32（32 位浮点数）等等，此外还可以自定义字节序。
简单说，ArrayBuffer对象代表原始的二进制数据，TypedArray 视图用来读写简单类型的二进制数据，DataView视图用来读写复杂类型的二进制数据。

这方面知识点不太常用，了解下，等到用的时候查查就是了。

# 其他
* [编程风格](http://es6.ruanyifeng.com/#docs/style) —— 主要参考了 [Airbnb](https://github.com/airbnb/javascript) 公司的 JavaScript 风格规范。
* [读懂 ECMAScript 规格](http://es6.ruanyifeng.com/#docs/spec) —— 对于规格的学习建议，阮一峰老师的建议如下：
> 规格文件是计算机语言的官方标准，详细描述语法规则和实现方法。
一般来说，没有必要阅读规格，除非你要写编译器。因为规格写得非常抽象和精炼，又缺乏实例，不容易理解，而且对于解决实际的应用问题，帮助不大。但是，如果你遇到疑难的语法问题，实在找不到答案，这时可以去查看规格文件，了解语言标准是怎么说的。规格是解决问题的“最后一招”。

# 最后
好啦~终于把下半部分写完了。有点虎头蛇尾，一开始写的东西很具体，到后来内容有点少。主要是因为前面部分我觉得是比较麻烦和常用的。写这篇博客主要是系统复习下ES6语法，简略地提一下各个语法的用法、注意点。大致知道了有些什么，以后遇到问题知道如何查资料如何解决就好了。
感觉自己写博客速度忒慢了，写ES6笔记断断续续花了我十个小时……
最后呢，还是那句话——**由自己整理写出博客的知识点才是真正牢牢掌握的知识点！**至此，我对ES6语法的理解加深了很多。看到此文的你可以去试试用写博客的方式来复习知识点哦~
希望我写的东西能帮助到一些朋友。

# 关于我
VioletJack，内驱工程师~专注于Vue前端相关的知识点整理、源码学习、内容分享。欢迎喜欢我文章的朋友关注我哦，我会努力产出优质内容~让我们始终相信：code change world!