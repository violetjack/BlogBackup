---
title: Vue.js 源码学习六 —— VNode虚拟DOM学习
date: 2018-02-22
tag: "Vue.js源码学习"
---

> 初六和家人出去玩，没写完博客。跳票了~

所谓虚拟DOM，是一个用于表示真实 DOM 结构和属性的 JavaScript 对象，这个对象用于对比虚拟 DOM 和当前真实 DOM 的差异化，然后进行局部渲染从而实现性能上的优化。在Vue.js 中虚拟 DOM 的 JavaScript 对象就是 VNode。
接下来我们一步步分析：

# VNode 是什么？
---
既然是虚拟 DOM 的作用是转为真实的 DOM，那这就是一个渲染的过程。所以我们看看 render 方法。在之前的学习中我们知道了，vue 的渲染函数 `_render` 方法返回的就是一个 VNode 对象。而在 `initRender` 初始化渲染的方法中定义的 `vm._c` 和 `vm.$createElement` 方法中，`createElement` 最终也是返回 VNode 对象。所以 VNode 是渲染的关键所在。
话不多说，来看看这个VNode为何方神圣。
```
// src/core/vdom/vnode.js
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functioanl scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag // 当前节点标签名
    this.data = data // 当前节点数据（VNodeData类型）
    this.children = children // 当前节点子节点
    this.text = text // 当前节点文本
    this.elm = elm // 当前节点对应的真实DOM节点
    this.ns = undefined // 当前节点命名空间
    this.context = context // 当前节点上下文
    this.fnContext = undefined // 函数化组件上下文
    this.fnOptions = undefined // 函数化组件配置项
    this.fnScopeId = undefined // 函数化组件ScopeId
    this.key = data && data.key // 子节点key属性
    this.componentOptions = componentOptions // 组件配置项 
    this.componentInstance = undefined // 组件实例
    this.parent = undefined // 当前节点父节点
    this.raw = false // 是否为原生HTML或只是普通文本
    this.isStatic = false // 静态节点标志 keep-alive
    this.isRootInsert = true // 是否作为根节点插入
    this.isComment = false // 是否为注释节点
    this.isCloned = false // 是否为克隆节点
    this.isOnce = false // 是否为v-once节点
    this.asyncFactory = asyncFactory // 异步工厂方法 
    this.asyncMeta = undefined // 异步Meta
    this.isAsyncPlaceholder = false // 是否为异步占位
  }

  // 容器实例向后兼容的别名
  get child (): Component | void {
    return this.componentInstance
  }
}
```
其实就是一个普通的 JavaScript Class 类，中间有各种数据用于描述虚拟 DOM，下面用一个例子来看看VNode 是如何表现 DOM 的。
```html
    <div id="app">
        <span>{{ message }}</span>
        <ul>
            <li v-for="item of list" class="item-cls">{{ item }}</li>
        </ul>
    </div>

    <script>
        new Vue({
            el: '#app',
            data: {
                message: 'hello Vue.js',
                list: ['jack', 'rose', 'james']
            }
        })
    </script>
```
这个例子最终结果如图：![HTML显示结果](http://upload-images.jianshu.io/upload_images/1987062-fa2929532dc88449.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
简化后的VNode对象结果如图：
```json
{
    "tag": "div",
    "data": {
        "attr": { "id": "app" }
    },
    "children": [
        {
            "tag": "span",
            "children": [
                { "text": "hello Vue.js" }
            ]
        },
        {
            "tag": "ul",
            "children": [
                {
                    "tag": "li",
                    "data": { "staticClass": "item-cls" },
                    "children": [
                        { "text": "jack" }
                    ]
                },
                {
                    "tag": "li",
                    "data": { "staticClass": "item-cls" },
                    "children": [
                        { "text": "rose" }
                    ]
                },
                {
                    "tag": "li",
                    "data": { "staticClass": "item-cls" },
                    "children": [
                        { "text": "james" }
                    ]
                }
            ]
        }
    ],
    "context": "$Vue$3",
    "elm": "div#app"
}
```
在看VNode的时候小结以下几点：
* 所有对象的 `context` 选项都指向了 Vue 实例。
* `elm` 属性则指向了其相对应的真实 DOM 节点。
* DOM 中的文本内容被当做了一个只有 `text` 没有 `tag` 的节点。
* 像 class、id 等HTML属性都放在了 `data` 中

我们了解了VNode 是如何描述 DOM 之后，来学习如何将虚拟
 DOM 变为真实的 DOM。

# patch —— Virtual DOM 的核心
---
从之前的文章中可以知道，Vue的渲染过程（无论是初始化视图还是更新视图）最终都将走到 `_update` 方法中，再来看看这个 `_update` 方法。

```js
  // src/core/instance/lifecycle.js
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    
    if (!prevVnode) {
      // 初始化渲染
      vm.$el = vm.__patch__(
        vm.$el, vnode, hydrating, false /* removeOnly */,
        vm.$options._parentElm,
        vm.$options._refElm
      )
      // no need for the ref nodes after initial patch
      // this prevents keeping a detached DOM tree in memory (#5851)
      vm.$options._parentElm = vm.$options._refElm = null
    } else {
      // 更新渲染
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```
不难发现更新试图都是使用了 `vm.__patch__` 方法，我们继续往下跟。
```js
// src/platforms/web/runtime/index.js
Vue.prototype.__patch__ = inBrowser ? patch : noop
```
这里啰嗦一句，要找vue的全局方法，如 `vm.aaa` ,直接查找 `Vue.prototype.aaa` 即可。
继续找下去：
```
// src/platforms/web/runtime/patch.js
export const patch: Function = createPatchFunction({ nodeOps, modules })
```
找到 `createPatchFunction` 方法~
```
// src/core/vdom/patch.js
export function createPatchFunction (backend) {
  ……
  return function patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
    // 当前 VNode 未定义、老的 VNode 定义了，调用销毁钩子。
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // 老的 VNode 未定义，初始化。
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue, parentElm, refElm)
    } else {
      // 当前 VNode 和老 VNode 都定义了，执行更新操作
      // DOM 的 nodeType http://www.w3school.com.cn/jsref/prop_node_nodetype.asp
      const isRealElement = isDef(oldVnode.nodeType) // 是否为真实 DOM 元素
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        // 修改已有根节点
        patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
      } else {
        // 已有真实 DOM 元素，处理 oldVnode
        if (isRealElement) {
          // 挂载一个真实元素，确认是否为服务器渲染环境或者是否可以执行成功的合并到真实 DOM 中
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              // 调用 insert 钩子
              // inserted：被绑定元素插入父节点时调用 
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            }
          }
          // 不是服务器渲染或者合并到真实 DOM 失败，创建一个空节点替换原有节点
          oldVnode = emptyNodeAt(oldVnode)
        }

        // 替换已有元素
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // 创建新节点
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // 递归更新父级占位节点元素，
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // 销毁旧节点
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }
    // 调用 insert 钩子
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```
具体解析看代码注释~抛开调用生命周期钩子和销毁就节点不谈，我们发现代码中的关键在于 `createElm` 和 `patchVnode` 方法。

## createElm

先看 `createElm` 方法，这个方法创建了真实 DOM 元素。
```
  function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested // for transition enter check
    // 创建组件
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    if (isDef(tag)) {
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

      createChildren(vnode, children, insertedVnodeQueue)
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue)
      }
      insert(parentElm, vnode.elm, refElm)
    } else if (isTrue(vnode.isComment)) {
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }
```
重点关注代码中的方法执行。代码太多，就不贴出来了，简单说说用途。
* `cloneVNode` 用于克隆当前 vnode 对象。
* `createComponent` 用于创建组件，在调用了组件初始化钩子之后，初始化组件，并且重新激活组件。在重新激活组件中使用 `insert` 方法操作 DOM。
* `nodeOps.createElementNS` 和 `nodeOps.createElement` 方法，其实是真实 DOM 的方法。
* `setScope` 用于为 scoped CSS 设置作用域 ID 属性
* `createChildren` 用于创建子节点，如果子节点是数组，则遍历执行 `createElm` 方法，如果子节点的 text 属性有数据，则使用 `nodeOps.appendChild(...)` 在真实 DOM 中插入文本内容。
* `insert` 用于将元素插入真实 DOM 中。

所以，这里的 `nodeOps` 指的肯定就是真实的 DOM 节点了。最终，这些所有的方法都调用了 `nodeOps` 中的方法来操作 DOM 元素。

> 这里顺便科普下 DOM 的[属性和方法](http://www.w3school.com.cn/jsref/dom_obj_all.asp)。下面把源码中用到的几个方法列出来便于学习：
> * appendChild: 向元素添加新的子节点，作为最后一个子节点。
> * insertBefore: 在指定的已有的子节点之前插入新节点。
> * tagName: 返回元素的标签名。
> * removeChild: 从元素中移除子节点。
> * createElementNS: 创建带有指定命名空间的元素节点。
> * createElement: 创建元素节点。
> * createComment: 创建注释节点。
> * createTextNode: 创建文本节点。
> * setAttribute: 把指定属性设置或更改为指定值。
> * nextSibling: 返回位于相同节点树层级的下一个节点。
> * parentNode: 返回元素父节点。
> * setTextContent: 获取文本内容（这个未在w3school中找到，不过应该就是这个意思了）。
 
OK，知道以上方法就比较好理解了，`createElm` 方法的最终目的就是创建真实的 DOM 对象。

## patchVnode

看过了创建真实 DOM 后，我们来学习虚拟 DOM 如何实现 DOM 的更新。这才是虚拟 DOM 的存在意义 —— 比对并局部更新 DOM 以达到性能优化的目的。
看代码~
```js
  // 补丁 vnode
  function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
    // 新旧 vnode 相等
    if (oldVnode === vnode) {
      return
    }

    const elm = vnode.elm = oldVnode.elm
    // 异步占位
    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // 如果新旧 vnode 为静态；新旧 vnode key相同；
    // 新 vnode 是克隆所得；新 vnode 有 v-once 的属性
    // 则新 vnode 的 componentInstance 用老的 vnode 的。
    // 即 vnode 的 componentInstance 保持不变。
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    // 执行 data.hook.prepatch 钩子。
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      // 遍历 cbs，执行 update 方法
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      // 执行 data.hook.update 钩子
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    // 旧 vnode 的 text 选项为 undefined
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        // 新旧 vnode 都有 children，且不同，执行 updateChildren 方法。
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        // 清空文本，添加 vnode
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 移除 vnode
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // 如果新旧 vnode 都是 undefined，清空文本
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 有不同文本内容，更新文本内容
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      // 执行 data.hook.postpatch 钩子，表明 patch 完毕
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```
源码中添加了一些注释便于理解，来理一下逻辑。
1. 如果两个vnode相等，不需要 patch，退出。
2. 如果是异步占位，执行 hydrate 方法或者定义 isAsyncPlaceholder 为 true，然后退出。
3. 如果两个vnode都为静态，不用更新，所以讲以前的 componentInstance 实例传给当前 vnode，并退出。
4. 执行 prepatch 钩子。
5. 遍历调用 update 回调，并执行 update 钩子。
6. 如果两个 vnode 都有 children，且 vnode 没有 text、两个 vnode 不相等，执行 updateChildren 方法。这是虚拟 DOM 的关键。
7. 如果新 vnode 有 children，而老的没有，清空文本，并添加 vnode 节点。
8. 如果老 vnode 有 children，而新的没哟，清空文本，并移除 vnode 节点。
9. 如果两个 vnode 都没有 children，老 vnode 有 text ，新 vnode 没有 text ，则清空 DOM 文本内容。
10. 如果老 vnode 和新 vnode 的 text 不同，更新 DOM 元素文本内容。
11. 调用 postpatch 钩子。

其中，`addVnodes` 方法和 `removeVnodes` 都比较简单，很好理解。这里我们来看看关键代码 `updateChildren` 方法。
```js
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly 是一个只用于 <transition-group> 的特殊标签，
    // 确保移除元素过程中保持一个正确的相对位置。
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        // 开始老 vnode 向右一位
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        // 结束老 vnode 向左一位
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 新旧开始 vnode 相似，进行pacth。开始 vnode 向右一位
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 新旧结束 vnode 相似，进行patch。结束 vnode 向左一位
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        // 新结束 vnode 和老开始 vnode 相似，进行patch。
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
        // 老开始 vnode 插入到真实 DOM 中，老开始 vnode 向右一位，新结束 vnode 向左一位
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        // 老结束 vnode 和新开始 vnode 相似，进行 patch。
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
        // 老结束 vnode 插入到真实 DOM 中，老结束 vnode 向左一位，新开始 vnode 向右一位
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        // 获取老 Idx 的 key
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        // 给老 idx 赋值
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) {
          // 如果老 idx 为 undefined，说明没有这个元素，创建新 DOM 元素。
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          // 获取 vnode
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            // 如果生成的 vnode 和新开始 vnode 相似，执行 patch。
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
            // 赋值 undefined，插入 vnodeToMove 元素
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // 相同的key不同的元素，视为新元素
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        // 新开始 vnode 向右一位
        newStartVnode = newCh[++newStartIdx]
      }
    }
    // 如果老开始 idx 大于老结束 idx，如果是有效数据则添加 vnode 到新 vnode 中。
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      // 移除 vnode
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }
```
表示已看晕……让我们慢慢捋一捋……
1.  看参数，其中 oldCh 和 newCh 即表示了新旧 vnode 数组，两组数组通过比对的方式来差异化更新 DOM。
2. 定义了一些变量：开始索引值、结束索引值、开始vnode、结束vnode等等……
3. 进行循环遍历，遍历条件为 oldStartIdx <= oldEndIdx 和 newStartIdx <= newEndIdx，在遍历过程中，oldStartIdx 和 newStartIdx 递增，oldEndIdx 和 newEndIdx 递减。当条件不符合跳出遍历循环。
4. 如果 oldStartVnode 和 newStartVnode 相似，执行 patch。
![image](http://upload-images.jianshu.io/upload_images/1987062-3c53cb4442d3fc58?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5. 如果 oldEndVnode 和 newEndVnode 相似，执行 patch。
6. 如果 oldStartVnode 和 newEndVnode 相似，执行 patch，并且将该节点移动到 vnode 数组末一位。
![image](http://upload-images.jianshu.io/upload_images/1987062-0b47f3cb7f762873?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
7. 如果 oldEndVnode 和 newStartVnode 相似，执行 patch，并且将该节点移动到 vnode 数组第一位。
![image](http://upload-images.jianshu.io/upload_images/1987062-f6203babe1e15791?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
8. 如果没有相同的 idx，执行 createElm 方法创建元素。
9. 如果如有相同的 idx，如果两个 vnode 相似，执行 patch，并且将该节点移动到 vnode 数组第一位。如果两个 vnode 不相似，视为新元素，执行 createElm 创建。
![image](http://upload-images.jianshu.io/upload_images/1987062-2a6b908889782ac4?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
10. 如果老 vnode 数组的开始索引大于结束索引，说明新 node 数组长度大于老 vnode 数组，执行 addVnodes 方法添加这些新 vnode 到 DOM 中。
![image](http://upload-images.jianshu.io/upload_images/1987062-d353c99c30bb5f25?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
11. 如果老 vnode 数组的开始索引小于结束索引，说明老 node 数组长度大于新 vnode 数组，执行 removeVnodes 方法从 DOM 中移除老 vnode 数组中多余的 vnode。
![image](http://upload-images.jianshu.io/upload_images/1987062-c8aa456d7f2839da?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

嗯……就是这样！

# 最后
毕竟是Vue的核心功能之一，虽然省略了不少代码，但博客篇幅很长。写了两天才写完。不过写完博客后感觉对于 Vue 的理解又加深了很多。
在下一篇博客中，我们一起来学习template的解析。

# 参考文档
* [Vue官网](https://cn.vuejs.org/)
* [VirtualDOM与diff(Vue实现)](https://github.com/answershuto/learnVue/blob/master/docs/VirtualDOM%E4%B8%8Ediff(Vue%E5%AE%9E%E7%8E%B0).MarkDown)
* [Vue2.1.7源码学习](http://hcysun.me/2017/03/03/Vue%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0/)
* [w3school](http://www.w3school.com.cn/index.html)

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