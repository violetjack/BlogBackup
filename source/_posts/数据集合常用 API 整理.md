---
title: 常用数据集合 API 整理
date: 2018-05-10
---

> 最近刷 LeetCode 题总是遇到 Array、Object、Set、Map 这类数据结构，虽然知道有些什么 API，但是每次用总是要查查 MDN 才放心。非常浪费时间，所以这里好好整理下这些数据集合的常用 API。

# Array

* Array.length 返回数组长度
* Array.from() 方法从一个类似数组或可迭代对象中创建一个新的数组实例。传入类似数组对象及map回调方法，返回新数组。
* fill() 方法用一个固定值填充一个数组中从起始索引到终止索引内的全部元素。返回修改后的数组（原数组）。
* forEach() 方法对数组的每个元素执行一次提供的函数。
* reduce() 方法对累加器和数组中的每个元素（从左到右）应用一个函数，将其减少为单个值。
* includes() 方法用来判断一个数组是否包含一个指定的值，根据情况，如果包含则返回 true，否则返回false。
* join() 方法将一个数组（或一个[类数组对象](https://developer.mozilla.org/zh-CN//docs/Web/JavaScript/Guide/Indexed_collections#Working_with_array-like_objects)）的所有元素连接成一个字符串并返回这个字符串。
* keys() 方法返回一个新的Array迭代器，它包含数组中每个索引的键。
* map() 方法创建一个新数组，其结果是该数组中的每个元素都调用一个提供的函数后返回的结果。
* 数组操作：
  * concat() 方法用于合并两个或多个数组。此方法不会更改现有数组，而是**返回一个新数组**。
  * pop() 方法从数组中删除最后一个元素，并返回该元素的值。此方法更改数组的长度。
  * push() 方法将一个或多个元素添加到数组的末尾，并返回新数组的长度。
  * shift() 方法从数组中删除第一个元素，并返回该元素的值。此方法更改数组的长度。
  * unshift() 方法将一个或多个元素添加到数组的开头，并返回新数组的长度。
  * reverse() 方法将数组中元素的位置颠倒。
  * sort() 方法用[就地（ in-place ）的算法](https://en.wikipedia.org/wiki/Sorting_algorithm#Stability)对数组的元素进行排序，并返回数组。 sort 排序不一定是[稳定的](https://zh.wikipedia.org/wiki/%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95#.E7.A9.A9.E5.AE.9A.E6.80.A7)。默认排序顺序是根据字符串Unicode码点。会改变原数组，返回排序后的数组。
  * splice() 方法通过删除现有元素和/或添加新元素来更改一个数组的内容。可以用于删除也可用于插入数据。返回被删除的元素（集合）。
  * slice() 方法返回一个从开始到结束（不包括结束）选择的数组的一部分**浅拷贝到一个新数组对象。且原始数组不会被修改。**截取内容的范围是 start <= val < end。可以使用 arr.slice() 对数组进行浅拷贝。

# Object
* Object.create() 方法创建一个新对象，使用现有的对象来提供新创建的对象的__proto__。 可以通过 `Object.create(null)` 创建空对象。
* Object.defineProperty() 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性， 并返回这个对象。
* Object.freeze() 方法可以冻结一个对象，冻结指的是不能向这个对象添加新的属性，不能修改其已有属性的值，不能删除已有属性，以及不能修改该对象已有属性的可枚举性、可配置性、可写性。也就是说，这个对象永远是不可变的。该方法返回被冻结的对象。
* Object.seal() 方法封闭一个对象，阻止添加新属性并将所有现有属性标记为不可配置。当前属性的值只要可写就可以改变。
* Object.keys() 方法会返回一个由一个给定对象的自身可枚举属性组成的数组，数组中属性名的排列顺序和使用[`for...in`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/for...in "for...in语句以任意顺序遍历一个对象的可枚举属性。对于每个不同的属性，语句都会被执行。") 循环遍历该对象时返回的顺序一致 （两者的主要区别是 一个 for-in 循环还会枚举其原型链上的属性）。

# Set
* size 属性将会返回[`Set`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Set "Set 对象允许你存储任何类型的唯一值，无论是原始值或者是对象引用。")对象中元素的个数。
* add() 方法用来向一个 Set 对象的末尾添加一个指定的值。
* clear() 方法用来清空一个 Set 对象中的所有元素。
* delete() 方法可以从一个 Set 对象中删除指定的元素。
* forEach 方法根据集合中元素的顺序，对每个元素都执行提供的 callback 函数一次。
* has() 方法返回一个布尔值来指示对应的值value是否存在Set对象中

# Map
* size 可访问属性返回 [`Map`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Map "此页面仍未被本地化, 期待您的翻译!") 对象的元素数量。
* get() 方法用来获取一个 Map 对象中指定的元素。如果找不到返回 `undefined`
* set() 方法为 Map 对象添加一个指定键（key）和值（value）的新元素。
* clear() 方法会移除Map对象中的所有元素。
* delete() 方法用于移除 Map 对象中指定的元素。
* forEach() ?方法将会以插入顺序对 Map 对象中的每一个键值对执行一次参数中提供的回调函数。
* has() 方法返回一个布尔值，用来表明 map 中是否存在指定元素。

# 最后

先就这些啦~整理出来以免每次用到的时候都去查 MDN （我都查烦了），这种常用的整理出来记在脑子里比较好。


