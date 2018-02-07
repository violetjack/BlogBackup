---
title: ES6 学习笔记（上）
date: 2018-02-06 16:20:52
---

 > 认真学习了一遍ES6，发现很多很好用的功能。
学习资料：[《ECMAScript 6 入门》](http://es6.ruanyifeng.com/#README)

之前写JS，虽然也遇到一些ES6语法，基本就是理解就行，没有深入学习过。这次认真看了一遍ES6，发现里面有许多实用的东西。下面是对本书学习的一些笔记

# let 和 const 命令
解决var变量提升、变量全局的问题。

**let 和 const 都只作用域本块级作用域内。**
```
let a = 123
{
    let a = 456
    console.log(a)
}
console.log(a)

// 456
// 123
```

**let 和 const 定义的变量不能同名。**
**let 和 const 定义的变量不可以变量提升。**
```
a = 5
let a

// Error: a is not defined
```
**const定义的量内存不可变**
```
const a = 6
const obj = {
    name: 'violetjack'
}

// 以下写法会报错，因为他们指向了另一个内存地址
// a = 6
// obj = {}

obj.name = 'jerry'
obj.age = 56
console.log(obj)

// { name: 'jerry', age: 56 }
```

# 变量的结构与赋值
快速赋值，优化代码可读性。

**用法都差不多，一一对应即可**
```
let [a, b, c] = [1, 2, 3] // 数组结构
let { name:mName, age: mAge} = { name:'violetjack', age: 28 } // 对象结构
let [e,f,g] = 'mmp' // 字符串结构
console.log(a)
console.log(mName)
console.log(e + f + g)

// 1
// violetjack
// mmp
```

# 字符串扩展
提出了一些处理字符串的方式

**提供了一些处理大于 `0xFFFF` 的字符串的方法**
**字符串检索**
```
let s = 'Hello world!';

s.startsWith('Hello') // true
s.endsWith('!') // true
s.includes('o') // true

// includes()：返回布尔值，表示是否找到了参数字符串。
// startsWith()：返回布尔值，表示参数字符串是否在原字符串的头部。
// endsWith()：返回布尔值，表示参数字符串是否在原字符串的尾部。
```
**可以操作复制、填充字符串：`repeat`,`padStart`,`padEnd`**
**模板字符串，解决字符串拼接问题。**
```
const name = 'VioletJack'
const str = 'welcome to ' + name + "'s blog"
const str2 = `welcome to ${name}'s blog`
console.log(str)
console.log(str2)

// welcome to VioletJack's blog
// welcome to VioletJack's blog
```
相比于第一种方式，第二种方式可读性好很多。也避免了在使用 `'` `"` 两个符号的时候需要转译的步骤。同时在引用数据上使用 `${value}` 的方式引用。非常方便！

# 正则的扩展
对于正则我一向晕晕乎乎，没啥收获。关于正则我要另外写篇博客涨涨姿势。

# 数值的扩展
主要提供了一些数字高级算法。
**parseInt和parseFloat都在Number中调用，减少js中的全局方法。**
```
// ES5的写法
parseInt('12.34') // 12
parseFloat('123.45#') // 123.45

// ES6的写法
Number.parseInt('12.34') // 12
Number.parseFloat('123.45#') // 123.45
```
**求指数，通过 `**` 实现**
```
2 ** 2 // 2*2 = 4
2 ** 3 // 2*2*2 = 8
```
其他数学算法就不一一列举了，基本用不着，用到了百度即可。

# 函数的扩展
在函数参数传递方面实现了更多传递方式，实现了箭头函数。总体上是让函数应用上ES6的结构、`rest` 参数。此外还有几个提案，可以去书上看看。

**更多的参数传递方式**
```
// 函数参数默认值
function func01(a, b = 11) {
    console.log(a + b)
}

func01(15)
func01(10, undefined)
func01(1, 1)

// 参数解构
function func02({a = 6, b}) {
    console.log(a + b)
}

func02({a: 11, b: 12})
func02({b: 6})

// rest参数
function func03(...vals) {
    let count = 0
    for (let value of vals) {
        count += value
    }
    console.log(count)
}

func03(1, 2, 3, 4, 5)
func03(11, 22, 33, 44, 55)
```
**提出箭头函数，简化代码。**
```
var f = v => v;
// 等同于
var f = function(v) {
  return v;
};

var f = () => 5;
// 等同于
var f = function () { return 5 };

var sum = (num1, num2) => num1 + num2;
// 等同于
var sum = function(num1, num2) {
  return num1 + num2;
};
```
注意如果参数或者返回结果是对象，需要用`()`包裹。
```
var fun = ({ a, b }) => a + b
var fun02 = a => ({ value: a })
```
另外一个特别要注意的就是 `this`，我看到过好几篇博客来解释箭头函数的 `this` 的，可见这个 `this` 的特殊性。可以[在掘金搜索箭头函数](https://juejin.im/search?query=%E7%AE%AD%E5%A4%B4%E5%87%BD%E6%95%B0)，里面全是解释（tu cao）箭头函数的~
下面搬运书上对箭头函数的注意点。

> 箭头函数有几个使用注意点。
（1）函数体内的this对象，就是定义时所在的对象，而不是使用时所在的对象。
（2）不可以当作构造函数，也就是说，不可以使用new命令，否则会抛出一个错误。
（3）不可以使用arguments对象，该对象在函数体内不存在。如果要用，可以用 rest 参数代替。
（4）不可以使用yield命令，因此箭头函数不能用作 Generator 函数。
上面四点中，第一点尤其值得注意。this对象的指向是可变的，但是在箭头函数中，它是固定的。

# 数组的扩展
**扩展运算符，和函数中的rest参数是一回事。我的理解就是把数组拆分成一个个值。**
```
let arr = [1, 2, 3, 4]
console.log(...arr)
console.log([0, 8, 6, ...arr, 10])
let str = 'jack'
console.log(...str)

// 1 2 3 4
// [ 0, 8, 6, 1, 2, 3, 4, 10 ]
// j a c k
```
**Array.from 将类数组对象转为真正的数组。**
**Array.of 将多个值转为数组**
```
Array.of(3, 11, 8) 
// [3,11,8]
```
**copyWithin 将数组中某些内容来替换指定内容**
**find 和 findIndex 用于找到数组中第一个符合条件的值和索引位置。**
```
var a = [1, 4, -5, 10].find((n) => n < 2)
console.log(a)
var b = [1, 5, 10, 15].findIndex(function (value, index, arr) {
    return value > 11;
}) 
console.log(b)

// 1
// 3
```
**fill 用于填充数组**
**entries、keys和values用来遍历数组非常方便，通过不同方法返回所需数组内容。**
```
for (let index of ['a', 'b'].keys()) {
  console.log(index);
}
// 0
// 1

for (let elem of ['a', 'b'].values()) {
  console.log(elem);
}
// 'a'
// 'b'

for (let [index, elem] of ['a', 'b'].entries()) {
  console.log(index, elem);
}
// 0 "a"
// 1 "b"
```
**includes 方法与字符串的 includes 类似，用于查找数组中是否有某个值，返回布尔类型的值。**
**ES6中将数组空位转为undefined**
```
console.log([...['a',,'b']])
// [ "a", undefined, "b" ]
```

# 对象的扩展
**对象属性的简洁表示法，也就是我们常看到的ES6的写法。对象中不再必须要传递 `key-value` 的形式了。**
```
let a = 12
let b = 22
let obj = { a, b }
console.log(obj)

// { a: 12. b: 22 }

let obj02 = {
    log() {
        console.log('Hello!')
    }
}
```
**对于对象属性名，支持使用 `[字符串]` 的形式来作为对象属性名。**
```
let obj = {
    name: 'jack',
    ['age']: 28,
    ['se' + 'x']: 'male'
}
console.log(obj)

// {name: "jack", age: 28, sex: "male"}
```
**Object.is方法用于比较两个值是否严格相等。**
**Object.assign 用于对象的合并，如果有同名属性，后添加的覆盖之前的属性。**
```
let obj1 = {
    name: 'jack'
}

let obj2 = {
    ['age']: 28,
    ['se' + 'x']: 'male'
}

let obj3 = {
    job: 'js'
}

let obj4 = {
    name: 'rose',
    hobby: 'game'
}

Object.assign(obj1, obj2, obj3, obj4)
console.log(obj1)

// {name: "rose", age: 28, sex: "male", job: "js", hobby: "game"}
```
**对象的__proto__属性是一个内部属性，所以前后有下划线。不建议修改该属性。**
**ES6的super关键字用于继承，可以理解为java的super关键字。**
**Object.keys()，Object.values()，Object.entries()和Array的类似，是对对象的一个遍历过程。**
```
let {keys, values, entries} = Object;
let obj = { a: 1, b: 2, c: 3 };

for (let key of keys(obj)) {
  console.log(key); // 'a', 'b', 'c'
}

for (let value of values(obj)) {
  console.log(value); // 1, 2, 3
}

for (let [key, value] of entries(obj)) {
  console.log([key, value]); // ['a', 1], ['b', 2], ['c', 3]
}
```
**在对象中也可以使用扩展运算符 `...`**

# Symbol
这玩意的用处我不太理解，就是为了表示一个唯一的值？
**ES6 的7中原始数据类型：`Symbol`、`undefined`、`null`、`Boolean`、`String`、`Number`、`Object`**

# Set 和 Map 数据结构
**Set是一个构造函数，它的值都是唯一的。**
```
const s = new Set();

[2, 3, 5, 4, 5, 2, 2].forEach(x => s.add(x));

for (let i of s) {
  console.log(i);
}
// 2 3 5 4
```
**Set可用于去除数组中的重复项。**
```
const set = new Set([1, 2, 3, 4, 4]);
[...set]
// [1, 2, 3, 4]
```
**操作Set的语法如下**
```
add(value)：添加某个值，返回 Set 结构本身。
delete(value)：删除某个值，返回一个布尔值，表示删除是否成功。
has(value)：返回一个布尔值，表示该值是否为Set的成员。
clear()：清除所有成员，没有返回值。
```
**WeakSet与Set的不同点**
> WeakSet 的成员只能是对象，而不能是其他类型的值。
WeakSet 中的对象都是弱引用，即垃圾回收机制不考虑 WeakSet 对该对象的引用，也就是说，如果其他对象都不再引用该对象，那么垃圾回收机制会自动回收该对象所占用的内存，不考虑该对象还存在于 WeakSet 之中。

**Map不同于传统Object对象的地方在于，Object的属性名只能是String类型的。而Map可以有任意类型的属性名。**
**Map的key如果是一个对象，那么这个key应该指向同一个内存（可以用const定义之后再当做参数传入map作为属性名。）。**
```
// error
const map = new Map();

map.set(['a'], 555);
map.get(['a']) // undefined

// success
const map = new Map();

const k1 = ['a'];
const k2 = ['a'];

map
.set(k1, 111)
.set(k2, 222);

map.get(k1) // 111
map.get(k2) // 222
```
**WeakMap与Map的区别：**
> WeakMap只接受对象作为键名（null除外），不接受其他类型的值作为键名。
WeakMap的键名所指向的对象，不计入垃圾回收机制。

好吧，暂时先到这儿。另外一部分后面继续整理吧~知识点还是蛮多的。
