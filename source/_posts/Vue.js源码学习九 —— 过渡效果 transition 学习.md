---
title: Vue.js 源码学习九 —— 过渡效果 transition 学习
date: 2018-04-12
tag: "Vue.js源码学习"
---

> 在学习 element ui 时，发现组件的过渡用的是 Vue.js 提供的 <transition> 标签。这里来好好认识下 vue 的过渡到底是如何工作的。

# 简介
废话不多说，详细的内容请看[官方文档](https://cn.vuejs.org/v2/guide/transitions.html)，里面有详细的分析和例子够你看懂了（就是费时间~）。简单说说我对 vue 过渡的理解。经过一下午的折腾，总结出以下几点：

* **有四种情况会触发过渡效果：**
  1 v-if 
  2 v-show 
  3 动态组件（如 component 的 is 属性） 
  4 组件根节点发生变化（如 v-if v-else 切换根节点）
* **过渡效果 CSS 命名规律：**（name 属性，默认为 v）-（行为：enter、leave、appear、move）-（阶段：无、active、to）
* **有三种方式来设置过渡样式：**
  1 为 <transition> 标签设定 name 属性。
  2 在 <transition> 标签中插入 `enter-active-class` 等设置自定义过渡类名。
  3 使用 JavaScript 在过渡的[钩子](https://cn.vuejs.org/v2/guide/transitions.html#JavaScript-%E9%92%A9%E5%AD%90)处修改过渡样式。
* **个人理解：`<transition>` 标签用于单个元素的进入和离开效果。`<transition-group>` 标签用于处理如 `v-for` 遍历这样多个元素的过渡动画。**

# 自己实现个过渡方法

先来两个简单例子理解下 transition（为了节省篇幅和便于查看写在 JSFiddle 中）有兴趣的朋友可以看下~
**例1：[v-enter 和 v-leave 简单实现](https://jsfiddle.net/VioletJack/0pnqc99z/1/)**
**例2：[v-move 简单实现](https://jsfiddle.net/VioletJack/3takx674/)**

# transition 学习
## 1. 基本原理是什么？

基本原理还是 CSS3 的 `transition`、`transform`、`animation` 这几个属性。用户定义过渡效果，Vue.js 进行处理。下面我们通过 <transition> 过渡的进入的过程看一下：

* 插入元素
* 解析 <transition> 标签，获取对应的过渡类名。这里默认就 `v-` 开头了。
* 为元素定义 v-enter 和 v-enter-active 两个类。`class="v-enter v-enter-active"`。
* 下一帧移除 v-enter，添加 v-enter-to。`class="v-enter-active v-enter-to"`。
* 获取过渡时间，延时执行回调函数。
* 回调函数中移除 v-enter、v-enter-active 和 v-enter-to 的这些过渡类名，完成过渡。
* 在整个过程中调用了 `beforeEnterHook` 、`enterHook`、`afterEnterHook`、`enterCancelledHook` 四个函数，执行相应的[ JavaScript 钩子](https://cn.vuejs.org/v2/guide/transitions.html#JavaScript-钩子)。

下面是 `enter` 函数的代码及注释：
```js
// 进入过渡效果
export function enter (vnode: VNodeWithData, toggleDisplay: ?() => void) {
  const el: any = vnode.elm

  // call leave callback now 执行 leave 回调函数
  if (isDef(el._leaveCb)) {
    el._leaveCb.cancelled = true
    el._leaveCb()
  }

  // 解析 transition 的数据（class、tag、name等）
  const data = resolveTransition(vnode.data.transition)
  if (isUndef(data)) {
    return
  }

  /* istanbul ignore if */
  if (isDef(el._enterCb) || el.nodeType !== 1) {
    return
  }

  const {
    css,
    type,
    enterClass,
    enterToClass,
    enterActiveClass,
    appearClass,
    appearToClass,
    appearActiveClass,
    beforeEnter,
    enter,
    afterEnter,
    enterCancelled,
    beforeAppear,
    appear,
    afterAppear,
    appearCancelled,
    duration
  } = data

  // 将作为子组件的根节点放置时，我们需要检查 <transition> 的父元素是否出现检查。
  let context = activeInstance
  let transitionNode = activeInstance.$vnode
  while (transitionNode && transitionNode.parent) {
    transitionNode = transitionNode.parent
    context = transitionNode.context
  }

  const isAppear = !context._isMounted || !vnode.isRootInsert

  if (isAppear && !appear && appear !== '') {
    return
  }

  // 获取进入的 class
  // v-enter
  const startClass = isAppear && appearClass
    ? appearClass
    : enterClass
  // v-enter-active
  const activeClass = isAppear && appearActiveClass
    ? appearActiveClass
    : enterActiveClass
  // v-enter-to
  const toClass = isAppear && appearToClass
    ? appearToClass
    : enterToClass

  // 4个生命周期钩子函数
  const beforeEnterHook = isAppear
    ? (beforeAppear || beforeEnter)
    : beforeEnter
  const enterHook = isAppear
    ? (typeof appear === 'function' ? appear : enter)
    : enter
  const afterEnterHook = isAppear
    ? (afterAppear || afterEnter)
    : afterEnter
  const enterCancelledHook = isAppear
    ? (appearCancelled || enterCancelled)
    : enterCancelled

  // https://cn.vuejs.org/v2/guide/transitions.html#显性的过渡持续时间
  const explicitEnterDuration: any = toNumber(
    isObject(duration)
      ? duration.enter
      : duration
  )

  if (process.env.NODE_ENV !== 'production' && explicitEnterDuration != null) {
    checkDuration(explicitEnterDuration, 'enter', vnode)
  }

  const expectsCSS = css !== false && !isIE9
  const userWantsControl = getHookArgumentsLength(enterHook)

  // 完成进入过渡后的回调函数
  const cb = el._enterCb = once(() => {
    if (expectsCSS) {
      // 移除 v-enter-to 和 v-enter-active
      removeTransitionClass(el, toClass)
      removeTransitionClass(el, activeClass)
    }
    if (cb.cancelled) {
      if (expectsCSS) {
        // 移除 v-enter
        removeTransitionClass(el, startClass)
      }
      // 调用 enter-cancelled
      enterCancelledHook && enterCancelledHook(el)
    } else {
      afterEnterHook && afterEnterHook(el)
    }
    el._enterCb = null
  })

  if (!vnode.data.show) {
    // 通过注入一个 insert 钩子，将待处理的 leave 元素移除。
    mergeVNodeHook(vnode, 'insert', () => {
      const parent = el.parentNode
      const pendingNode = parent && parent._pending && parent._pending[vnode.key]
      if (pendingNode &&
        pendingNode.tag === vnode.tag &&
        pendingNode.elm._leaveCb
      ) {
        pendingNode.elm._leaveCb()
      }
      enterHook && enterHook(el, cb)
    })
  }

  // start enter transition
  beforeEnterHook && beforeEnterHook(el)
  // 预期 CSS
  if (expectsCSS) {
    // 添加 v-enter v-enter-active
    addTransitionClass(el, startClass)
    addTransitionClass(el, activeClass)
    // 下一帧
    nextFrame(() => {
      // 移除 v-enter
      removeTransitionClass(el, startClass)
      if (!cb.cancelled) {
        // 添加 v-enter-to
        addTransitionClass(el, toClass)
        if (!userWantsControl) {
          // 预期进入时间
          if (isValidDuration(explicitEnterDuration)) {
            setTimeout(cb, explicitEnterDuration)
          } else {
            // 当 transition 结束
            whenTransitionEnds(el, type, cb)
          }
        }
      }
    })
  }

  if (vnode.data.show) {
    toggleDisplay && toggleDisplay()
    enterHook && enterHook(el, cb)
  }

  if (!expectsCSS && !userWantsControl) {
    cb()
  }
}
```
## 2. [过渡的类名](https://cn.vuejs.org/v2/guide/transitions.html#过渡的类名)和[自定义过渡的类名](https://cn.vuejs.org/v2/guide/transitions.html#自定义过渡的类名)如何用于 <transition> 中？

在 [<transition>](https://cn.vuejs.org/v2/api/#transition) 中一共有如下属性（props）：
```js
export const transitionProps = {
  name: String,
  appear: Boolean,
  css: Boolean,
  mode: String,
  type: String,
  enterClass: String,
  leaveClass: String,
  enterToClass: String,
  leaveToClass: String,
  enterActiveClass: String,
  leaveActiveClass: String,
  appearClass: String,
  appearActiveClass: String,
  appearToClass: String,
  duration: [Number, String, Object]
}
```
可以看到其中就有这些自定义过渡类名，如 enterClass。这些属性如被传入到 <transition> 子组件的 data.transition 对象中。
```js
    // extractTransitionData 函数返回组件的所有 propsData 和 listener
    const data: Object = (child.data || (child.data = {})).transition = extractTransitionData(this)
```
而这个 data.transition 对象在 enter 函数中用到：
```js
 const data = resolveTransition(vnode.data.transition)
```
`resolveTransition` 函数：
```
// 解析 transition 过渡 CSS
export function resolveTransition (def?: string | Object): ?Object {
  if (!def) {
    return
  }
  // 合并过渡类名和自定义过渡类名
  if (typeof def === 'object') {
    const res = {}
    if (def.css !== false) {
      // 使用 name，默认为 v
      extend(res, autoCssTransition(def.name || 'v'))
    }
    extend(res, def)
    return res
  } else if (typeof def === 'string') {
    return autoCssTransition(def)
  }
}
// 通过 name 属性获取过渡 CSS 类名
const autoCssTransition: (name: string) => Object = cached(name => {
  return {
    enterClass: `${name}-enter`,
    enterToClass: `${name}-enter-to`,
    enterActiveClass: `${name}-enter-active`,
    leaveClass: `${name}-leave`,
    leaveToClass: `${name}-leave-to`,
    leaveActiveClass: `${name}-leave-active`
  }
})
```
resolveTransition 函数合并了过渡类名和自定义过渡类名，返回最终的过渡类名。之后就是使用这些类名来实现过渡动画。
*PS：从源码中可以知道自定义过渡类名要优先于 name 定义的过渡类名。*
小结一下就是：**Vue.js 通过 <transition> 的 props 获取自定义过渡类名，通过 <transition> 的 name 属性解析获取过渡类名，两者合并成为最终过渡类名，用以实现过渡效果。**

## 3. JavaScript 钩子如何实现？

从 `enter` 函数中可以知道，在特定时间点会调用指定 JavaScript 钩子函数，所以我们只需绑定好函数即可按时间点触发。像这样：
```js
enterHook && enterHook(el, cb)
```

## 4. transition 组件和 transition-group 标签的基本原理是什么？

其实就是 Vue.js 的组件，在其中实现了过渡效果而已。
transition 中只能包含一个子元素，标签通过 render 函数来渲染子元素（不渲染自身，所以我们在 DOM 中看不到 transition 节点）。主要用于控制元素的进入和离开，当元素离开后元素就从 DOM 中移除了。
transition-group 可以包含多个子元素，也是用 render 函数，渲染为指定标签名的元素。相比 transition 多了一个 v-move 属性用于控制多个组件间的移动速度。

## 5. v-if、v-show、component 等组件变化如何监听？

在使用 v-if、v-else 和 component 切换组件的时候，v-if、v-else 需要传入 key 以区分相同标签的不同元素。而 component 标签不需要。在代码中会解析 key 和 component 名组成新的 key，所以两个不同的 component 也会拥有不同的 key 实现切换效果。
```js
    var id = "__transition-" + (this._uid) + "-";
    child.key = child.key == null
      ? child.isComment
        ? id + 'comment'
        : id + child.tag
      : isPrimitive(child.key)
        ? (String(child.key).indexOf(id) === 0 ? child.key : id + child.key)
        : child.key;
```
而对于 v-show，做了特殊标记 —— 当有 v-show 指令时标记 child.data.show 为 true：
```js
    if (child.data.directives && child.data.directives.some(d => d.name === 'show')) {
      child.data.show = true
    }
```
之后再过渡的逻辑中对 v-show 做了些处理，实现过渡效果。
同时，在 `v-show` 的源码 `src/platforms/web/runtime/directives/show.js` 中对于 transition 也做了一些处理。比如在 update 方法中获取 transition，如果有过渡则 v-show 使用过渡效果，否则使用 `style.display` 来隐藏元素。
```js
  update (el: any, { value, oldValue }: VNodeDirective, vnode: VNodeWithData) {
    if (value === oldValue) return
    vnode = locateNode(vnode)
    // 过渡效果
    const transition = vnode.data && vnode.data.transition
    if (transition) {
      vnode.data.show = true
      if (value) {
        enter(vnode, () => {
          el.style.display = el.__vOriginalDisplay
        })
      } else {
        leave(vnode, () => {
          el.style.display = 'none'
        })
      }
    } else {
      // 隐藏
      el.style.display = value ? el.__vOriginalDisplay : 'none'
    }
  },
```

## 6. transition 中两个相同标签的组件为何要用 key 分开？

使用 key 和 tagName 来判断是否为同一个节点。
```js
function isSameChild (child: VNode, oldChild: VNode): boolean {
  return oldChild.key === child.key && oldChild.tag === child.tag
}
```

## 8. 过渡逻辑和过渡组件如何作用于一起

在源码中有四个过渡相关的源码：
* `src/platforms/web/runtime/components/transition.js` <transition> 组件源码。
* `src/platforms/web/runtime/components/transition-group.js` <transition-group> 组件源码
* `src/platforms/web/runtime/transition-util.js` 过渡工具代码。
* `src/platforms/web/runtime/modules/transition.js` 过渡逻辑代码。

前三个很好理解，最后一个 transition.js 其实是在 patch 方法中和 v-show 中使用的~
```js
// src/platforms/web/runtime/directives/show.js
import { enter, leave } from '../modules/transition'
```
v-show 中调用了 transition 的 enter 和 leave 函数，在 v-show 作用于过渡效果时调用。
另外一个使用的地方比较隐蔽，先来看看 transition.js 导出的内容：
```js
// src/platforms/web/runtime/modules/transition.js
function _enter (_: any, vnode: VNodeWithData) {
  if (vnode.data.show !== true) {
    enter(vnode)
  }
}

export default inBrowser ? {
  create: _enter,
  activate: _enter,
  remove (vnode: VNode, rm: Function) {
    /* istanbul ignore else */
    if (vnode.data.show !== true) {
      leave(vnode, rm)
    } else {
      rm()
    }
  }
} : {}
```
这里讲 enter 和 leave 函数在方法中使用并导出（如果是浏览器的话）。继续往下找：
```js
// src/platforms/web/runtime/modules/index.js
import transition from './transition'
```
导入到 modules 文件夹 index.js，index.js 在 patch.js 中使用了。
```js
// src/platforms/web/runtime/patch.js
import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

const modules = platformModules.concat(baseModules)
export const patch: Function = createPatchFunction({ nodeOps, modules })
```
在此处合并 modules，并且创建了 patch 方法。这个 patch 方法在之前写的[Vue.js 源码学习六 —— VNode虚拟DOM学习](https://www.jianshu.com/p/9db8eb16d76f)中提到过，用于对比虚拟 DOM，实现差异化更新。
可以看下 modules 在 createPatchFunction 方法中做了些什么？
```js
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  ...
}
```
这里可以发现 transition.js 中导出的 create、activate 和 remove 方法都是 patch 的生命周期函数。也就是说当元素创建、激活、移除行为时就会执行 transition.js 中的逻辑，而 <transition> 和 <transition-group> 组件都会有组件的这些行为。Vue.js 很巧妙的将组件相关行为都交给了 patch 的生命周期去处理，学习了！

## 8. 过渡模式 mode 的实现原理是啥

在 <transition> 组件的 render 函数中有这么一段：
```js
      // 控制离开/进入的过渡时间序列。有效的模式有 "out-in" 和 "in-out"；默认同时生效。
      if (mode === 'out-in') {
        // return placeholder node and queue update when leave finishes
        this._leaving = true
        mergeVNodeHook(oldData, 'afterLeave', () => {
          this._leaving = false
          this.$forceUpdate()
        })
        return placeholder(h, rawChild)
      } else if (mode === 'in-out') {
        if (isAsyncPlaceholder(child)) {
          return oldRawChild
        }
        let delayedLeave
        const performLeave = () => { delayedLeave() }
        mergeVNodeHook(data, 'afterEnter', performLeave)
        mergeVNodeHook(data, 'enterCancelled', performLeave)
        mergeVNodeHook(oldData, 'delayLeave', leave => { delayedLeave = leave })
      }
```
这里就是 mode 的实现代码了，先看看两种 mode 的用法

> * in-out：新元素先进行过渡，完成之后当前元素过渡离开。
> * out-in：当前元素先进行过渡，完成之后新元素过渡进入。

可以看到，在 out-in 逻辑中，当切换元素时，先不渲染第二个组件而是返回，之后才会返回 placeholder 函数结果，当第一个元素完全 leave 后加载第二个元素。而在 in-out 元素中做的是将第一个元素延时到第二个元素 enter 后再 leave。

## 9. <transition-group> 的 v-move 重新排序一组内容，如何实现的移动变化？

比如我们有一个 1-5 的数组使用 v-for 遍历显示到 transition-group 中，当数组发生变化时，会做如下操作：

* 初始数组 `[ 1, 2, 3, 4, 5 ]`
* 数组发生变化 `[ 1, 4, 3, 2, 5 ]`
* 在 render 函数中记录变化前后额数组 preChildren 和 children 两个 VNode 数组。
* 在 render 函数中使用 [getBoundingClientRect()](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect) 方法记录变化前每个元素的位置 oldPos。
* 获取要保留和移除的元素数组。
* 渲染变化后的数组元素。
* 在 beforeUpdate 方法中使用 patch 方法移除要移除的元素。
* 进入 Updated 方法中，注意此时渲染结果已经是新数组 `[ 1, 4, 3, 2, 5 ]` 了。
* 获取过渡类名和子元素数组 children。
* 遍历调用
  * 执行回调函数
  * 计算当前各元素位置 newPos
  * 根据 oldPos 和 newPos，使用内联样式 `translate(${dx}px,${dy}px)` 将元素移动到之前的位置，看着就像是  `[ 1, 2, 3, 4, 5 ]`
* 最后遍历元素，添加 moveClass 类名，移除 `translate(${dx}px,${dy}px)` 内联样式。绑定 [transitionend](https://developer.mozilla.org/en-US/docs/Web/Events/transitionend) 事件。

代码太长，就不多贴了~可以**[点击这里](https://github.com/violetjack/VueStudyDemos/blob/master/VueCodes/vue/src/platforms/web/runtime/components/transition-group.js)**跳转查看。**总结下来就是先改变元素，然后把元素移动成之前的样子，然后使用过渡类名定义过渡时间实现过渡效果。**
v-move 的关键就是“假装元素位置没变”的行为。让我们看上去像是慢慢移动的。
```js
function applyTranslation (c: VNode) {
  const oldPos = c.data.pos
  const newPos = c.data.newPos
  const dx = oldPos.left - newPos.left
  const dy = oldPos.top - newPos.top
  if (dx || dy) {
    // 定义 0 秒的 translate 内联样式把元素移动到原来的样子
    c.data.moved = true
    const s = c.elm.style
    s.transform = s.WebkitTransform = `translate(${dx}px,${dy}px)`
    s.transitionDuration = '0s'
  }
}
```

## 10. vue 的 transition 和 CSS3 的 transition 有何不同？

基本原理都是使用了 CSS3 的 transition，但是 Vue 的 transition 组件是配合着 VDOM 来写的、同时提供了过渡各阶段效果的 CSS 和 JS 控制，便于我们快捷、精确、安全地实现一些简单或复杂的过渡效果。

# 最后

原本只是想看看 transition 如何实现，却扯出这么一堆问题。其中关于 transition 和 transition-group 组件讲的有点草率，有兴趣可以再深入学习下~
从本次学习中我学到了：

* 更加优雅高效的 JS 逻辑写法（patch 中的生命周期统一处理 DOM 操作中的逻辑）
* 更加熟悉 CSS3 的 transition 过渡属性。
* 解决了我对 transition 的各种疑问。

OK，关于 Vue 的过渡效果就聊到这儿了，写了三天……我得去休息休息了 0.0