---
title: Vue.js 源码学习七 —— template 解析过程学习
date: 2018-03-04
tag: "Vue.js源码学习"
---

> 这次，来学习下Vue是如何解析HTML代码的。

# template 解析用在哪

从之前学习 Render 的过程中我们知道，template 的编译在 `$mount` 方法中出现过。
```js
// src/platforms/web/entry-runtime-with-compiler.js
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          // 首字母为#号，看作是ID。
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        // 为真实 DOM，直接获取html
        template = template.innerHTML
      } else {
        return this
      }
    } else if (el) {
      // 获取 HTML
      template = getOuterHTML(el)
    }
    if (template) {
      // 进行编译并赋值给 vm.$options
      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      // 渲染函数
      options.render = render
      // 静态渲染方法
      options.staticRenderFns = staticRenderFns
    }
  }
  return mount.call(this, el, hydrating)
}
```
其实以上代码总结起来就4步：

1. 获取el元素。
2. 判断el是否为body或者html。
3. 为$options编译render函数。
4. 执行之前的mount函数。

关键在于第三步，编译 render 函数上。先获取 template，即获取HTML内容，然后执行 compileToFunctions 来编译，最后将 render 和 staticRenderFns 传给 vm.$options 对象。
顺便看看这两个方法都用在哪里？
```js
  // src/core/instance/render.js
  Vue.prototype._render = function (): VNode {
    try {
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`)
    }
    return vnode
  }
```
```js
// src/core/instance/render-helpers/render-static.js
export function renderStatic (
  index: number,
  isInFor: boolean
): VNode | Array<VNode> {
  const cached = this._staticTrees || (this._staticTrees = [])
  let tree = cached[index]
  if (tree && !isInFor) {
    return tree
  }
  // otherwise, render a fresh tree.
  tree = cached[index] = this.$options.staticRenderFns[index].call(
    this._renderProxy,
    null,
    this 
  )
  markStatic(tree, `__static__${index}`, false)
  return tree
}
```
由此可见，template 编译生成的方法都用在了渲染行为中。

# 编译 template 的整体逻辑

下面我们顺着编译代码往下找。在 mount 方法中执行了 `compileToFunctions ` 方法。
```js
const { render, staticRenderFns } = compileToFunctions(template, {
  shouldDecodeNewlines,
  shouldDecodeNewlinesForHref,
 delimiters: options.delimiters,
 comments: options.comments
}, this)
```
找到方法的所在之处：
```js
// src/platforms/web/compiler/index.js
const { compile, compileToFunctions } = createCompiler(baseOptions)
```
```js
// src/compiler/index.js
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 将template转为AST语法树对象
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    // 优化
    optimize(ast, options)
  }
  // 生成渲染代码
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```
先看里面的 baseCompile 方法，其作用为将 HTML 字符串转为 AST 抽象语法树对象，并进行优化，最后生成渲染代码。返回值中 render 为渲染字符串，staticRenderFns 为渲染字符串数组。
之后再来看看 createCompilerCreator 方法：
```js
// src/compiler/create-compiler.js
export function createCompilerCreator (baseCompile: Function): Function {
  return function createCompiler (baseOptions: CompilerOptions) {
    function compile (
      template: string,
      options?: CompilerOptions
    ): CompiledResult {
      const finalOptions = Object.create(baseOptions)
      const errors = []
      const tips = []
      finalOptions.warn = (msg, tip) => {
        (tip ? tips : errors).push(msg)
      }

      if (options) {
        // merge custom modules
        if (options.modules) {
          finalOptions.modules =
            (baseOptions.modules || []).concat(options.modules)
        }
        // merge custom directives
        if (options.directives) {
          finalOptions.directives = extend(
            Object.create(baseOptions.directives || null),
            options.directives
          )
        }
        // copy other options
        for (const key in options) {
          if (key !== 'modules' && key !== 'directives') {
            finalOptions[key] = options[key]
          }
        }
      }
      // 执行传入的编译方法，并返回结果对象
      const compiled = baseCompile(template, finalOptions)
      if (process.env.NODE_ENV !== 'production') {
        errors.push.apply(errors, detectErrors(compiled.ast))
      }
      compiled.errors = errors
      compiled.tips = tips
      return compiled
    }

    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}
```
来看 compile 方法：合并 option 配置参数，然后执行外部传入的 baseCompile 方法，返回方法执行的返回结果。最终返回 `{ compile, compileToFunctions }`，
createCompileToFunctionFn 代码如下：
```js
export function createCompileToFunctionFn (compile: Function): Function {
  // 定义缓存
  const cache = Object.create(null)

  return function compileToFunctions (
    template: string,
    options?: CompilerOptions,
    vm?: Component
  ): CompiledFunctionResult {
    options = extend({}, options)
    const warn = options.warn || baseWarn
    delete options.warn

    // 确认缓存，有缓存直接返回
    const key = options.delimiters
      ? String(options.delimiters) + template
      : template
    if (cache[key]) {
      return cache[key]
    }

    // compile
    const compiled = compile(template, options)

    // turn code into functions
    const res = {}
    const fnGenErrors = []
    // 生成 render 和 staticRenderFns 方法
    res.render = createFunction(compiled.render, fnGenErrors)
    res.staticRenderFns = compiled.staticRenderFns.map(code => {
      return createFunction(code, fnGenErrors)
    })
    // 返回方法并缓存
    return (cache[key] = res)
  }
}
```
这里就找到了我们在 mount 方法中看到的 render 和 staticRenderFns 方法了。createCompileToFunctionFn 方法其实就是将传入的 render 和 staticRenderFns 字符串转为真实方法。

至此，捋一下思路：
template的编译用于render渲染行为中，所以template最后生成渲染函数。
template 的解析过程中

* 通过 baseCompile 方法进行编译；
* 通过 createCompilerCreator 中的 compile 方法合并配置参数并返回 baseCompile 方法执行结果；
* createCompilerCreator 返回 compile 方法和 compileToFunctions 方法；
* compileToFunctions 方法用于将方法字符串生成真实方法。

其实 `const { compile, compileToFunctions } = createCompiler(baseOptions)` 就是 createCompilerCreator 的返回结果。所以，在 mount 中使用的 compileToFunctions 方法就是 createCompileToFunctionFn 方法生成的。

![逻辑图](http://upload-images.jianshu.io/upload_images/1987062-61c14118fb4d28d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# baseCompile
整体思路滤清了，来看看关键的 baseCompile 方法。该方法进行了三步操作：
* parse 将HTML解析为 AST 元素。
* optimize 渲染优化。
* generate 解析成基本的 render 函数。

## parse
先来讲讲AST抽象语法树。维基百科的解释是：

> 在计算机科学中，抽象语法树（abstract syntax tree或者缩写为AST），或者语法树（syntax tree），是源代码的抽象语法结构的树状表现形式，这里特指编程语言的源代码。

parse 方法的最终目的就是将 template 解析为 AST 元素对象。在 parse 解析方法中，用到了大量的正则。正则的具体用法之前写过一篇文章：[一起来理解正则表达式](https://www.jianshu.com/p/0734cc319aa3)。代码量很多，考虑了各种解析的情况。这里不赘述太多，找一条主线来学习，其他内容我将在[项目](https://github.com/violetjack/VueStudyDemos/tree/master/VueCodes/vue)中注释。

来看看 parse 方法。
```js
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  // 定义了各种参数和方法
  parseHTML(template, {
    warn,
    expectHTML: options.expectHTML,
    isUnaryTag: options.isUnaryTag,
    canBeLeftOpenTag: options.canBeLeftOpenTag,
    shouldDecodeNewlines: options.shouldDecodeNewlines,
    shouldDecodeNewlinesForHref: options.shouldDecodeNewlinesForHref,
    shouldKeepComment: options.comments,
    start (tag, attrs, unary) {},
    end () {}
    chars (text: string) {},
    comment (text: string) {}
  )
  return root
}
```
实际上 parse 就是 parseHTML 的过程，最后返回AST元素对象。其中，传入的 options 配置对象中，start、end、chars、comment方法都会在 parseHTML 方法中用到。其实类似于生命周期钩子，在某个阶段执行。
parseHTML 方法是正则解析HTML的过程，这部分我将在之后的博客中单独说下，也可以看项目的注释，将不定时更新项目注释。

## optimize
该方法只是做了些标记静态节点的行为，目的是为了在重新渲染时不重复渲染静态节点，以达到性能优化的目的。
```js
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // 标记所有非静态节点
  markStatic(root)
  // 标记静态根节点
  markStaticRoots(root, false)
}
```

## generate
generate 方法用于将 AST 元素生成 render 渲染字符串。
```js
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```
最后生成如下这样的渲染字符串：
```
with(this){return _c('div',{attrs:{"id":"app"}},[_c('button',{on:{"click":hey}},[_v(_s(message))])])}
```
其中的 _c _v _s 等方法在哪里呢~这个我们之前说起过:
```js
// src/core/instance/render.js
// 创建vnode元素
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
// src/core/instance/render-helper/index.js
export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
}
```

# 最后
其实template部分真的内容展开超级多，之后会展开细说。原本计划大前天就把博客写出来的，结果看代码看着看着绕进去了。所以，还是那句话，看代码得抓住主线，带着问题去看，不要在意细枝末节。
这也算是我的经验教训了，以后每次看代码，牢记待着明确的问题去看去解决。想一次看懂整个项目的代码是不可行的。
下期预告，parseHTML 细节解析

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
