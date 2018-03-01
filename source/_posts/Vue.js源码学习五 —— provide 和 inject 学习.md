---
title: Vue.js 源码学习五 —— provide 和 inject 学习
date: 2018-02-20
tag: "Vue.js源码学习"
---

> 早上好！继续开始学习Vue源码吧~

在 Vue.js 的 `2.2.0+` 版本中添加加了 provide 和 inject 选项。他们成对出现，用于父级组件向下传递数据。
下面我们来看看源码~

# 源码位置
和之前一样，初始化的方法都是在 Vue 的 `_init` 方法中的。
```js
  // src/core/instance/init.js
  Vue.prototype._init = function (options?: Object) {
    ……
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
  }
```
这里找到 `initInjections` 和 `initProvide` 方法，这就是 `provide` 和 `inject` 的初始化方法了。这两个方法都是在 `src/core/instance/inject.js` 中。

# provide
> provide 选项应该是一个对象或返回一个对象的函数。该对象包含可注入其子孙的属性。在该对象中你可以使用 ES2015 Symbols 作为 key，但是只在原生支持 Symbol 和 Reflect.ownKeys 的环境下可工作。

先看源码：
```js
// src/core/instance/inject.js
export function initProvide (vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```
provide 是向下传递数据的选项。这里先拿到 provide 选项中的内容，如果有 provide 选项，将 provide 选项传递给 `vm._provided` 变为 Vue 实例全局数据。
这里看一下例子更清楚，下例中传入数据 `foo`，数据内容为 `bar`。
```js
var Provider = {
  provide: {
    foo: 'bar'
  },
  // ...
}
```

# inject
> inject 选项应该是一个字符串数组或一个对象，该对象的 key 代表了本地绑定的名称，value 为其 key (字符串或 Symbol) 以在可用的注入中搜索。

源码
```js
// src/core/instance/inject.js
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    observerState.shouldConvert = false
    Object.keys(result).forEach(key => {
      defineReactive(vm, key, result[key])
    })
    observerState.shouldConvert = true
  }
}
```
简化后的源码可以看到，首先通过 `resolveInject` 方法获取 inject 选项搜索结果，如果有搜索结果，遍历搜索结果并为其中的数据添加 setter 和 getter。
接着来看下 `resolveInject` 方法：
```js
export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    // inject 是 :any 类型因为流没有智能到能够指出缓存
    const result = Object.create(null)
    // 获取 inject 选项的 key 数组
    const keys = hasSymbol
      ? Reflect.ownKeys(inject).filter(key => {
        /* istanbul ignore next */
        return Object.getOwnPropertyDescriptor(inject, key).enumerable
      })
      : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = inject[key].from
      let source = vm
      while (source) {
        if (source._provided && provideKey in source._provided) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        } else if (process.env.NODE_ENV !== 'production') {
          warn(`Injection "${key}" not found`, vm)
        }
      }
    }
    return result
  }
}
```
获取 inject 选项的 key 数组，遍历 key 数组，通过向上冒泡来查找 provide 中是否有 key 与 inject 选项中 from 属性同名的，如果有，则将这个数据传递给 result；如果没有，检查 inject 是否有 default 选项设定默认值或者默认方法，如果有则将默认值返传给 result，最终返回 result 对象。
所以，inject 的写法应该是有 default 默认值的：
```js
const Child = {
  inject: {
    foo: { default: 'foo' }
  }
}
```
或者是有 from 查找键和 default 默认值的：
```js
const Child = {
  inject: {
    foo: {
      from: 'bar',
      default: 'foo'
    }
  }
}
```
或者为 default 默认值设定一个工厂方法：
```js
const Child = {
  inject: {
    foo: {
      from: 'bar',
      default: () => [1, 2, 3]
    }
  }
}
```
好吧，我承认这就是引用的官网的三个例子~ 不过意思到就好啦。
这里我有个疑问，既然在源码中主动去识别了 from 和 default，官网上说是
> 在 `2.5.0+` 的注入可以通过设置默认值使其变成可选项：

那么如下写法还可用吗？
```js
var Child = {
  inject: ['foo'],
  created () {
    console.log(this.foo) // => "bar"
  }
  // ...
}
```
为此，我们去查查 `2.2.0` 版本的Vue是怎么写的？
```js
export function initInjections (vm: Component) {
  const provide = vm.$options.provide
  const inject: any = vm.$options.inject
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
  if (inject) {
    // inject is :any because flow is not smart enough to figure out cached
    // isArray here
    const isArray = Array.isArray(inject)
    const keys = isArray
      ? inject
      : hasSymbol
        ? Reflect.ownKeys(inject)
        : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = isArray ? key : inject[key]
      let source = vm
      while (source) {
        if (source._provided && source._provided[provideKey]) {
          vm[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
    }
  }
}
```
从中可以看到，在这个版本 provide 和 inject 是一起初始化的。之后，将 provide 传给 vm._provide ，在获取 inject 选项的时候代码判断了 inject 是否为数组，如果是数组直接遍历数组，之后查找 provide 的代码差不多。
所以我推测： **在 `2.5.0+` 之后不能再使用数组形式的 inject 来搜索 provide 了。**
PS：这里没有去代码验证，如有问题，欢迎指出，谢谢！

# 最后
至此，provide 和 inject 的源码学习完毕啦~ 如果有任何问题和建议，欢迎联系我！谢谢！
预告：明天学习 Vue 的 VDOM、VNode 相关知识。欢迎继续关注~

# Vue.js学习系列
鉴于前端知识碎片化严重，我希望能够系统化的整理出一套关于Vue的学习系列博客。

# Vue.js学习系列项目地址
本文源码已收入到GitHub中，以供参考，当然能留下一个star更好啦^-^。
[https://github.com/violetjack/VueStudyDemos](https://github.com/violetjack/VueStudyDemos)

# 关于作者
VioletJack，高效学习前端工程师，喜欢研究提高效率的方法，也专注于Vue前端相关知识的学习、整理。
欢迎关注、点赞、评论留言~我将持续产出Vue相关优质内容。

新浪微博： http://weibo.com/u/2640909603
掘金：https://gold.xitu.io/user/571d953d39b0570068145cd1
CSDN: http://blog.csdn.net/violetjack0808
简书： http://www.jianshu.com/users/54ae4af3a98d/latest_articles
Github： https://github.com/violetjack

