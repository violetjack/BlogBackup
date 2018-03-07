---
title: Vue.js 源码学习八 —— HTML解析细节学习
date: 2018-03-04
tag: "Vue.js源码学习"
---

从上一篇博客中，我们知道了template编译的整体逻辑和template编译后用在了哪里。本文着重讲下HTML的解析过程。

# parse 方法

所有解析的起点就在 parse 方法中，parse方法最终将返回为一个 AST 语法树元素。
```js
// src/core/compiler/parser/index.js
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  warn = options.warn || baseWarn

  platformIsPreTag = options.isPreTag || no
  platformMustUseProp = options.mustUseProp || no
  platformGetTagNamespace = options.getTagNamespace || no

  transforms = pluckModuleFunction(options.modules, 'transformNode')
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
  postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')

  delimiters = options.delimiters

  const stack = []
  const preserveWhitespace = options.preserveWhitespace !== false
  let root
  let currentParent
  let inVPre = false
  let inPre = false
  let warned = false

  function warnOnce(msg){...}
  function closeElement(element){...}
  parseHTML(...)

  return root
}
```
可以看到，除了 `parseHTML`  方法外，其他都是定义变量、方法的行为。因此只需深入看 parseHTML 行为就好。
于是我们在 `src/core/compiler/parser/html-parser.js` 文件中找到 parseHTML 方法。

# parseHTML 中的几个方法

在源码中可以看到，parseHTML 中有四个方法，我们来一一解读。

## advance

```js
  // 推进。向前推进n个字符
  function advance (n) {
    index += n
    html = html.substring(n)
  }
```
将index的值向后移动n位，然后从第n个字符开始截取 HTML 内容字符串。

# parseStartTag
```js
  // 解析开始标签
  function parseStartTag () {
    const start = html.match(startTagOpen)
    if (start) {
      const match = {
        tagName: start[1],
        attrs: [],
        start: index
      }
      advance(start[0].length)
      let end, attr
      while (!(end = html.match(startTagClose)) && (attr = html.match(attribute))) {
        advance(attr[0].length)
        match.attrs.push(attr)
      }
      if (end) {
        match.unarySlash = end[1]
        advance(end[0].length)
        match.end = index
        return match
      }
    }
  }
```
该方法使用正则匹配获取HTML开始标签，并且将开始标签中的属性都保存到一个数组中。最终返回标签结果：标签名、标签属性和标签起始结束位置。例如标签为 `<button v-on:click="hey">` 返回结果如下：
```json
    {
        "attrs": [
            [
                " v-on:click='hey'",
                "v-on:click",
                "=",
                "hey",
                "undefined",
                "undefined",
            ]
        ],
        "end": 48,
        "start": 23,
        "tagName": "button",
        "unarySlash": ""
    }
```

## handleStartTag
```js
  // 处理开始标签，将开始标签中的属性提取出来。
  function handleStartTag (match) {
    const tagName = match.tagName
    const unarySlash = match.unarySlash

    // 解析结束标签
    if (expectHTML) {
      if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
        parseEndTag(lastTag)
      }
      if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
        parseEndTag(tagName)
      }
    }

    const unary = isUnaryTag(tagName) || !!unarySlash

    // 解析开始标签的属性名和属性值
    const l = match.attrs.length
    const attrs = new Array(l)
    for (let i = 0; i < l; i++) {
      const args = match.attrs[i]
      // hackish work around FF bug https://bugzilla.mozilla.org/show_bug.cgi?id=369778
      if (IS_REGEX_CAPTURING_BROKEN && args[0].indexOf('""') === -1) {
        if (args[3] === '') { delete args[3] }
        if (args[4] === '') { delete args[4] }
        if (args[5] === '') { delete args[5] }
      }
      const value = args[3] || args[4] || args[5] || ''
      const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
        ? options.shouldDecodeNewlinesForHref
        : options.shouldDecodeNewlines
      attrs[i] = {
        name: args[1],
        value: decodeAttr(value, shouldDecodeNewlines)
      }
    }

    // 将标签及其属性推如堆栈中
    if (!unary) {
      stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs })
      lastTag = tagName
    }
    // 触发 options.start 方法。
    if (options.start) {
      options.start(tagName, attrs, unary, match.start, match.end)
    }
  }
```
该方法用于处理开始标签。如果是可以直接结束的标签，直接解析结束标签；然后遍历查找属性的属性值 value 传入数组；将开始标签的标签名、小写标签名、属性值传入堆栈中；将当前标签变为最后标签；最后触发 options.start 方法。
最后推入堆栈的数据如下
```json
    {
        "tag": "button",
        "lowerCasedTag": "button",
        "attrs": [
            { 
                "name": "v-on:click",
                "value": "hey"
            }
        ]
    }
```

## parseEndTag

```js
  // 解析结束TAG
  function parseEndTag (tagName, start, end) {
    let pos, lowerCasedTagName
    if (start == null) start = index
    if (end == null) end = index

    if (tagName) {
      lowerCasedTagName = tagName.toLowerCase()
    }

    // 找到同类的开始 TAG 在堆栈中的位置
    if (tagName) {
      for (pos = stack.length - 1; pos >= 0; pos--) {
        if (stack[pos].lowerCasedTag === lowerCasedTagName) {
          break
        }
      }
    } else {
      // If no tag name is provided, clean shop
      pos = 0
    }

    // 对堆栈中的大于等于 pos 的开始标签使用 options.end 方法。
    if (pos >= 0) {
      // Close all the open elements, up the stack
      for (let i = stack.length - 1; i >= pos; i--) {
        if (process.env.NODE_ENV !== 'production' &&
          (i > pos || !tagName) &&
          options.warn
        ) {
          options.warn(
            `tag <${stack[i].tag}> has no matching end tag.`
          )
        }
        if (options.end) {
          options.end(stack[i].tag, start, end)
        }
      }

      // Remove the open elements from the stack
      // 从栈中移除元素，并标记为 lastTag
      stack.length = pos
      lastTag = pos && stack[pos - 1].tag
    } else if (lowerCasedTagName === 'br') {
      // 回车标签
      if (options.start) {
        options.start(tagName, [], true, start, end)
      }
    } else if (lowerCasedTagName === 'p') {
      // 段落标签
      if (options.start) {
        options.start(tagName, [], false, start, end)
      }
      if (options.end) {
        options.end(tagName, start, end)
      }
    }
  }
```
解析结束标签。先是获取开始结束位置、小写标签名；然后遍历堆栈找到同类开始 TAG 的位置；对找到的 TAG 位置后的所有标签都执行 options.end 方法；将 pos 后的所有标签从堆栈中移除，并修改最后标签为当前堆栈最后一个标签的标签名；如果是br标签，执行 option.start 方法；如果是 p 标签，执行 options.start 和options.end 方法。（最后两个操作让我猜想 start 和 end 方法用于标签的开始和结束行为中。）

# parseHTML 的整体逻辑

之前所说的 options.start 等方法，其实在 parseHTML 的传参中传入的 start、end、chars、comment 这四个方法，这些方法会在parseHTML 方法特定的地方被使用，而这些方法中的逻辑下一节再讲。
这里先来看看在 parseHTML 方法的整体逻辑：
```js
// src/core/compiler/parser/html-parser.js
export function parseHTML (html, options) {
  const stack = []
  const expectHTML = options.expectHTML
  const isUnaryTag = options.isUnaryTag || no
  const canBeLeftOpenTag = options.canBeLeftOpenTag || no
  let index = 0
  let last, lastTag
  while (html) {
    last = html
    // 如果没有lastTag，并确保我们不是在一个纯文本内容元素中：script、style、textarea
    if (!lastTag || !isPlainTextElement(lastTag)) {
      // 文本结束，通过<查找。
      let textEnd = html.indexOf('<')
      // 文本结束位置在第一个字符，即第一个标签为<
      if (textEnd === 0) {
        // 注释匹配
        if (comment.test(html)) {
          const commentEnd = html.indexOf('-->')

          if (commentEnd >= 0) {
            // 如果需要保留注释，执行 option.comment 方法
            if (options.shouldKeepComment) {
              options.comment(html.substring(4, commentEnd))
            }
            advance(commentEnd + 3)
            continue
          }
        }

        // http://en.wikipedia.org/wiki/Conditional_comment#Downlevel-revealed_conditional_comment
        // 条件注释
        if (conditionalComment.test(html)) {
          const conditionalEnd = html.indexOf(']>')

          if (conditionalEnd >= 0) {
            advance(conditionalEnd + 2)
            continue
          }
        }

        // Doctype:
        const doctypeMatch = html.match(doctype)
        if (doctypeMatch) {
          advance(doctypeMatch[0].length)
          continue
        }

        // End tag: 结束标签
        const endTagMatch = html.match(endTag)
        if (endTagMatch) {
          const curIndex = index
          advance(endTagMatch[0].length)
          // 解析结束标签
          parseEndTag(endTagMatch[1], curIndex, index)
          continue
        }

        // Start tag: 开始标签
        const startTagMatch = parseStartTag()
        if (startTagMatch) {
          handleStartTag(startTagMatch)
          if (shouldIgnoreFirstNewline(lastTag, html)) {
            advance(1)
          }
          continue
        }
      }

      // < 标签位置大于等于0，即标签中有内容
      let text, rest, next
      if (textEnd >= 0) {
        // 截取从 0 - textEnd 的字符串
        rest = html.slice(textEnd)
        // 获取在普通字符串中的<字符，而不是开始标签、结束标签、注释、条件注释
        while (
          !endTag.test(rest) &&
          !startTagOpen.test(rest) &&
          !comment.test(rest) &&
          !conditionalComment.test(rest)
        ) {
          // < in plain text, be forgiving and treat it as text
          next = rest.indexOf('<', 1)
          if (next < 0) break
          textEnd += next
          rest = html.slice(textEnd)
        }
        // 最终截取字符串内容
        text = html.substring(0, textEnd)
        advance(textEnd)
      }

      if (textEnd < 0) {
        text = html
        html = ''
      }
      // 绘制文本内容，使用 options.char 方法。
      if (options.chars && text) {
        options.chars(text)
      }
    } else {
      // 如果lastTag 为 script、style、textarea
      let endTagLength = 0
      const stackedTag = lastTag.toLowerCase()
      const reStackedTag = reCache[stackedTag] || (reCache[stackedTag] = new RegExp('([\\s\\S]*?)(</' + stackedTag + '[^>]*>)', 'i'))
      const rest = html.replace(reStackedTag, function (all, text, endTag) {
        endTagLength = endTag.length
        if (!isPlainTextElement(stackedTag) && stackedTag !== 'noscript') {
          text = text
            .replace(/<!\--([\s\S]*?)-->/g, '$1') // <!--xxx--> 
            .replace(/<!\[CDATA\[([\s\S]*?)]]>/g, '$1') //<!CDATAxxx>
        }
        if (shouldIgnoreFirstNewline(stackedTag, text)) {
          text = text.slice(1)
        }
        // 处理文本内容，并使用 options.char 方法。
        if (options.chars) {
          options.chars(text)
        }
        return ''
      })
      index += html.length - rest.length
      html = rest
      // 解析结束tag
      parseEndTag(stackedTag, index - endTagLength, index)
    }

    // html文本到最后
    if (html === last) {
      // 执行 options.chars
      options.chars && options.chars(html)
      if (process.env.NODE_ENV !== 'production' && !stack.length && options.warn) {
        options.warn(`Mal-formatted tag at end of template: "${html}"`)
      }
      break
    }
  }

  // 清理所有残留标签
  parseEndTag()

  ...
}
```
具体的解析都写在注释里面了。
其实就是利用正则循环处理 html 文本内容，最后使用 advance 方法来截取后一段 html 文本。在解析过程中执行了 options 中的一些方法。
下面我们来看看传入的方法都做了些什么？

# parseHTML 传参的几个方法

## warn
```js
// src/core/compiler/parser/index.js
warn = options.warn || baseWarn
```
如果options中有 warn 方法，使用该方法。否则调用 baseWarn 方法。

## start
```js
    start (tag, attrs, unary) {
      // 确定命名空间
      const ns = (currentParent && currentParent.ns) || platformGetTagNamespace(tag)

      // 处理 IE 的 SVG bug
      if (isIE && ns === 'svg') {
        attrs = guardIESVGBug(attrs)
      }

      // 获取AST元素
      let element: ASTElement = createASTElement(tag, attrs, currentParent)
      if (ns) {
        element.ns = ns
      }

      if (isForbiddenTag(element) && !isServerRendering()) {
        element.forbidden = true
      }

      // 遍历执行 preTransforms 方法
      for (let i = 0; i < preTransforms.length; i++) {
        element = preTransforms[i](element, options) || element
      }

      // 处理各种方法
      if (!inVPre) {
        // v-pre
        processPre(element)
        if (element.pre) {
          inVPre = true
        }
      }
      if (platformIsPreTag(element.tag)) {
        inPre = true
      }
      if (inVPre) {
        // 处理原始属性
        processRawAttrs(element)
      } else if (!element.processed) {
        // v-for v-if v-once
        processFor(element)
        processIf(element)
        processOnce(element)
        // 元素填充？
        processElement(element, options)
      }

      // 检查根节点约束
      function checkRootConstraints (el) {
        if (process.env.NODE_ENV !== 'production') {
          if (el.tag === 'slot' || el.tag === 'template') {
            warnOnce(
              `Cannot use <${el.tag}> as component root element because it may ` +
              'contain multiple nodes.'
            )
          }
          if (el.attrsMap.hasOwnProperty('v-for')) {
            warnOnce(
              'Cannot use v-for on stateful component root element because ' +
              'it renders multiple elements.'
            )
          }
        }
      }

      // 语法树树管理
      if (!root) {
        // 无root
        root = element
        checkRootConstraints(root)
      } else if (!stack.length) {
        // 允许有 v-if, v-else-if 和 v-else 的根元素
        if (root.if && (element.elseif || element.else)) {
          checkRootConstraints(element)
          // 添加 if 条件
          addIfCondition(root, {
            exp: element.elseif,
            block: element
          })
        } else if (process.env.NODE_ENV !== 'production') {
          warnOnce(
            `Component template should contain exactly one root element. ` +
            `If you are using v-if on multiple elements, ` +
            `use v-else-if to chain them instead.`
          )
        }
      }
      if (currentParent && !element.forbidden) {
        // v-else-if v-else
        if (element.elseif || element.else) {
          // 处理 if 条件
          processIfConditions(element, currentParent)
        } else if (element.slotScope) { // slot-scope
          currentParent.plain = false
          const name = element.slotTarget || '"default"'
          ;(currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
        } else {
          // 将元素插入 children 数组中
          currentParent.children.push(element)
          element.parent = currentParent
        }
      }
      if (!unary) {
        currentParent = element
        stack.push(element)
      } else {
        // 关闭元素
        closeElement(element)
      }
    },
```
其实start方法就是处理 element 元素的过程。确定命名空间；创建AST元素 element；执行预处理；定义root；处理各类 v- 标签的逻辑；最后更新 root、currentParent、stack 的结果。
其中关键点在于 createASTElement 方法。可以看到该方法传递了 tag、attrs和currentParent。其中前两个参数是不是很熟悉？就是我们在 parseHTML 的 handleStartTag 方法中传给堆栈数组中的数据对象。
```json
    {
        "tag": "button",
        "lowerCasedTag": "button",
        "attrs": [
            { 
                "name": "v-on:click",
                "value": "hey"
            }
        ]
    }
```
最终通过 createASTElement 方法定义了一个新的 AST 对象。
```js
// 创建AST元素
export function createASTElement (
  tag: string,
  attrs: Array<Attr>,
  parent: ASTElement | void
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    parent,
    children: []
  }
}
```

## end
```js
    end () {
      // 删除尾随空格
      const element = stack[stack.length - 1]
      const lastNode = element.children[element.children.length - 1]
      if (lastNode && lastNode.type === 3 && lastNode.text === ' ' && !inPre) {
        element.children.pop()
      }
      // 退栈
      stack.length -= 1
      currentParent = stack[stack.length - 1]
      // 关闭元素
      closeElement(element)
    },
```
end方法就很简单了，就是一个清理结束的过程。
从这里可以看到，stack中存的是个有序的数组，数组最后一个值永远是父级元素；currentParent表示当前的父级元素。其实也很好理解，收集HTML元素的时候是从最外层元素向内收集的，处理HTML内容的时候是从最内部元素向外处理的。所以，当最内部元素处理完后，将元素从对线中移除，开始处理当前最内部的元素。

## chars
```js
    chars (text: string) {
      if (!currentParent) {
        return
      }
      // IE textarea placeholder bug
      if (isIE &&
        currentParent.tag === 'textarea' &&
        currentParent.attrsMap.placeholder === text
      ) {
        return
      }
      // 获取元素 children
      const children = currentParent.children
      // 获取文本内容
      text = inPre || text.trim()
        ? isTextTag(currentParent) ? text : decodeHTMLCached(text)
        // only preserve whitespace if its not right after a starting tag
        : preserveWhitespace && children.length ? ' ' : ''
      if (text) {
        let res
        // inVPre 是判断 v-pre 的
        if (!inVPre && text !== ' ' && (res = parseText(text, delimiters))) {
          // 表达式，会转为 _s(message) 表达式
          children.push({
            type: 2,
            expression: res.expression,
            tokens: res.tokens,
            text
          })
        } else if (text !== ' ' || !children.length || children[children.length - 1].text !== ' ') {
          // 纯文本内容
          children.push({
            type: 3,
            text
          })
        }
      }
    },
```
chars方法用来处理非HTML标签的文本。如果是表达式，通过 parseText 方法解析文本内容并传递给当前元素的 children；如果是普通文本直接传递给当前元素的 children。

## comment
```js
    comment (text: string) {
      currentParent.children.push({
        type: 3,
        text,
        isComment: true
      })
    }
```
comment方法用来保存需要保存在语法树中的注释。它与保存普通文本类似，只是多了 `isComment: true`。

# 生成语法树
我这里写了个demo，并且抓取了AST元素最后生成结果。
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hey</title>
    <script src="vue.js"></script>
</head>
<body>
    <div id="app">
        <!-- this is vue parse demo -->
        <button v-on:click="hey">{{ message }}</button>
        <span>你好！</span>
    </div>

    <script>
        new Vue({
            el: "#app",
            data: {
                message: "Hey Vue.js"
            },
            methods: {
                hey() {
                    this.message = "Hey Button"
                }
            }
        })
    </script>
</body>
</html>
```
结果如下：
![AST对象](http://upload-images.jianshu.io/upload_images/1987062-d7068bf059b179ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 最后
最后整理理一下思路:
* parseHTML 中的方法用于处理HTML开始和结束标签。
* parseHTML 方法的整体逻辑是用正则判断各种情况，进行不同的处理。其中调用到了 options 中的自定义方法。
* options 中的自定义方法用于处理AST语法树，最终返回出整个AST语法树对象。

可以这么说，parseHTML 方法中仅仅是使用正则解析 HTML 的行为，options 中的方法则用于自定义方法和处理 AST 语法树对象。

OK！HTML的解析部分就讲解完啦~配合着之前的那篇[学习Vue中那些正则表达式](https://www.jianshu.com/p/0734cc319aa3)，顺着我的思路，相信一定可以顺利GET解析过程的。

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