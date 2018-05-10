---
title: 被人忽视的 DOM API
date: 2018-03-25
---

> 在框架盛行的年代，还有多少人记得在没有框架时我们如何控制 dom 的行为呢？作者本人也一直忽视了这方面的学习，直到面试问到这个问题，下决心好好认识认识这个 dom api。

# node、document 和 element
在学习 dom api 时对这三者还是挺混乱的。理一下他们之间的关系。
## node
node 是一个接口，像 document 和 element 都是继承这个接口的。这个接口提供了 dom 节点的获取和操作方法。
node 有许多类型，下图列出了一些 node 的类型码。由图可见 element 的类型码为 1，文本节点类型码为 3，注释节点类型码为 8，document 的类型码为 9。

![node type](https://upload-images.jianshu.io/upload_images/1987062-2ff52ed17c0e5c1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这让我想到了 vue.js 中出现多次的代码：`if (child.nodeType === 3) { ... }` 其实就是判断当前节点是否为文本节点。

## element

从上面的内容可知，element 就是一个特殊的 node（nodeType == 1），其实 element 就是 HTML 各类标签，如 <p><div> 这类有特殊含义的，能够携带一些特殊属性的节点。所以说 element 可以用 node 的所有 api。

## document

同理，document 也是一个特殊的 node，它与 element 的不同之处在于 document 通常是 DOM 节点，即包含有 head 和 body 元素的一个 node。

参考：[Difference between Node object and Element object? - Stack Overflow](https://stackoverflow.com/questions/9979172/difference-between-node-object-and-element-object)

# api 学习

首先我发现一点：**所有 dom 操作的起点都是使用 document 去获取各种类型的 node （集合）然后再去执行各类 dom 操作的行为！**
由于 document 操作的 api 真的很多，所以我选取我想了解的部分学习了。在这里我学习 DOM API 的目的是：

> 学习 `document` 对象操作 dom 的方式，拥有脱离框架（jquery、vue等）来操作 dom 的能力。

## 获取节点

* **Document.documentElement** 返回 document 的直属后代元素。
* **Document.activeElement** 返回当前正在操作的元素
* **Document.body** 返回当前文档的 `<body>` 元素。与此类似的还有 Document.head 和 Document.scripts 两个属性返回当前文档的 `<head>` 和 `<script>` 元素。
* **Document.getElementByClassName()** 返回有给定样式名的元素列表
* **Document.getElementByTagName()** 返回有给定标签名的元素列表
* **document.getElementById()** 返回一个对识别元素的对象引用
* **document.querySelector()** 返回文档中第一个匹配指定选择器的元素
* **document.querySelectorAll()** 返回一个匹配指定选择器的元素节点列表
* **Node.childNodes** 返回一个包含了该节点所有子节点的实时的 `NodeList` 是“实时的”意思是，如果该节点的子节点发生了变化，`NodeList` 对象就会自动更新。
* **Node.firstChild & Node.lastChild** 返回该节点的第一个子节点或最后一个子节点，如果该节点没有子节点则返回 null。
* **Node.previousSibling & Node.nextSibling** 返回与该节点同级的上一个或下一个节点，如果没有返回null。
* **Node.ownerDocument** 返回这个元素属于的 `Document` 对象 。
* **Node.parentNode** 返回一个当前结点 `Node` 的父节点 。
* **Node.parentElement** 返回一个当前节点的父节点 `Element`。

## 操作节点

* **Document.createComment()** 创建一个新的注释节点并返回它
* **Document.createDocumentFragment()** 创建一个新的文档片段
* **Document.createElement()** 用给定的标签名创建一个新的元素。
* **Document.createTextNode()** 创建一个文字节点
* **Document.write()** 向文档中写入内容（与之有类似功能的是 Document.writeln() 不同之处在于后面多了个换行符。）
* **Element.innerHTML** 设置或返回元素的内容
* **Node.textContent** 获取或设置一个标签内所有子结点及其后代的文本内容。
* **Node.appendChild()** 向元素添加新的子节点，作为最后一个子节点。
* **Node.cloneNode()** 克隆元素（方法中传参为deep，如果deep为true则深拷贝。）
* **Node.insertBefore()** 在指定已有节点前插入新节点（没有 `insertAfter` 方法。可以使用 `insertBefore` 方法和 `nextSibling` 来模拟它。）
* **Node.normalize()** 合并元素中相邻文本节点
* **Node.removeChild()** 从元素中移除子节点
* **Node.replaceChild()** 替换元素中的子节点

其中 Document 的 `createXXX` 方法还有一些其他不常用的，如需使用请查阅 MDN。

## 其他常用属性和方法

* **Element.classList** 返回元素的 class 集合。
* **EventTaget.addEventListener()** 注册监听事件
* **Node.nodeType** 返回该节点的类型码
* **Node.nodeValue** 返回或设置当前节点的值。
* **Node.compareDocumentPosition()** 比较当前节点与任意文档中的另一个节点的位置关系。
* **Node.contains()** 传入的节点是否为该节点的后代节点。
* **Node.hasChildNodes()** 是否拥有子节点
* **Node.isEqualNode()** 检查两个元素是否相等
* **Node.isSameNode()** 检查两个元素是否为相同的节点


以上内容均参考了 MDN 上的内容：
* Node https://developer.mozilla.org/zh-CN/docs/Web/API/Node
* Element https://developer.mozilla.org/zh-CN/docs/Web/API/Element
* Document https://developer.mozilla.org/zh-CN/docs/Web/API/Document

# 写个操作 DOM 的例子

接下来就使用这些 API 来进行一些 DOM操作。

## 获取各个位置的节点。
这里写了个小demo：
```html
<div id="container">
    <div>
        <h1>get dom</h1>
        <ul id="list">
            <li><span>hello world 1</span></li>
            <li><span>hello world 2</span></li>
            <li><span>hello world 3</span></li>
            <li><span>hello world 4</span></li>
            <li><span>hello world 5</span></li>
        </ul>
    </div>
    <br/>
    <div>
        <button>commit</button>
    </div>
</div>

<script>
    var container = document.getElementById('container')
    console.log('列出所有node', container.childNodes)
    var h1 = document.getElementsByTagName('h1')[0]
    console.log('获取h1后的元素', h1.nextSibling)
    var uldiv = container.firstChild
    while (uldiv && uldiv.nodeType != 1) {
        uldiv = uldiv.nextSibling
    }
    var ul = uldiv.lastChild
    while (ul && ul.nodeType == 3) {
        ul = ul.previousSibling
    }
    console.log('获取ul中第一个元素内容', ul.firstChild)
    var doc = h1.ownerDocument
    console.log('获取当前 Document 对象', doc)
    var li1 = ul.firstChild
    console.log('获取li的父级节点', li1.parentElement)
    var button = document.getElementsByTagName("button")[0]
    button.onclick = log
    button.focus()
    button.click()
    console.log('获取正在操作的元素', document.activeElement)

    function log(){
        console.log('button is clicked')
    }
</script>
```
最后返回结果如图：
![返回结果](https://upload-images.jianshu.io/upload_images/1987062-9f315855a37c0b17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由于在 chrome 中空格、换行算是文本节点。所以获取最后元素的时候总是会获取到那些文本节点上去。这个要注意的。所以我在代码中使用 `nodeType == 1` 来区分是否为元素。
在上面例子中查找了各种关系的元素，解决日常元素获取问题应该不难了。

## 实践创建node、插入node和删除node。

```html
<div>
    <div id="container">
        <h2 id="child">Hello Child</h2>
    </div>
    <br/>
    <div id="buttonGroup"></div>
</div>

<script>
    // 父节点向子节点插入元素
    function appendChild(){
        var container = document.getElementById("container")
        var text = document.createElement("h2")
        text.textContent = 'Hello New Child'
        container.appendChild(text)
    }
    // 子节点获取父节点，在父节点后插入元素
    function appendParent(){
        var child = document.getElementById('child')
        var parent = child.parentElement
        var text = document.createElement("h1")
        text.textContent = 'Hello Parent'
        var root = parent.parentElement

        root.insertBefore(text, parent.nextSibling)
    }
    // 在当前元素前插入元素
    function appendPre(){
        var child = document.getElementById('child')
        var text = document.createElement("h2")
        text.textContent = 'Hello Pre Child'
        child.parentElement.insertBefore(text, child)
    }
    // 在当前元素后插入元素
    function appendNext(){
        var child = document.getElementById('child')
        var text = document.createElement("h2")
        text.textContent = 'Hello Next Child'
        child.parentElement.insertBefore(text, child.nextSibling)
    }
    // 移除父元素中最后一个子元素
    function removeEle(){
        var container = document.getElementById('container')
        if (container.lastChild) {
            container.removeChild(container.lastChild)
        }
    }
    // 替换父元素中的子元素
    function replaceEle(){
        var child = document.getElementById("child")
        var newNode = document.createElement('div')
        newNode.innerHTML = "<button>button</button>hello new node replaced"
        var parent = child.parentElement
        parent.replaceChild(newNode, child)
    }
    // 创建按钮组
    var ButtonGroup = document.getElementById("buttonGroup")
    var EventList = [ 
        "appendChild", 
        "appendParent", 
        "appendPre", 
        "appendNext", 
        "removeEle", 
        "replaceEle" 
    ]

    var ButtonArr = []
    for (var key of EventList) {
        var btn = document.createElement('button')
        btn.textContent = key
        btn.onclick = eval(key)
        ButtonArr.push(btn)
    }
    for (var b of ButtonArr) {
        ButtonGroup.appendChild(b)
    }
</script>
```
以上代码实现了**在各个位置插入元素**和**元素的删除替换**，点击[此处](https://jsfiddle.net/VioletJack/g4nt26yf/1/)查看运行结果。

## 简单实现 v-for、v-text、v-html、v-on 和 v-model 这些功能。

好吧，作为一个 Vue.js 爱好者，绕不开的想到了 Vue.js 操作 DOM 的一些功能。这里就试着简单实现下（不涉及 Virtual DOM，只是单纯的 DOM 修改）。
如果对 Vue 命令不了解可以去官网看看这些[指令](https://cn.vuejs.org/v2/api/#%E6%8C%87%E4%BB%A4)的用法。
```html
<html>
    <head>
        <meta charset="UTF-8">
        <title>Hello Ele</title>
    </head>
    <body>
        <div id="container">
            <h1>v-text</h1>
            <span>{{ message }}</span>
            <h1>v-html</h1>  
            <div v-html="messagespan"></div>          
            <h1>v-model</h1>
            <input v-model="message"/>
            <h1>v-on</h1>
            <input id="myInput" v-on:blur="blur" v-on:focus="focus"/>
            <h1>v-for</h1>
            <ul></ul>
        </div>
        
        <script>
            // v-text
            var message = "Hello World"
            var messagespan = "<span>Hello World</span>"
          
            var spans = document.getElementsByTagName("span")
            for (var span of spans) {
                if (span.textContent == "{{ message }}") {
                    span.textContent = message
                }
            }
            // v-html
            var container = document.getElementById("container")
            var divs = container.getElementsByTagName("div")
            for(var div of divs){
                if (div.getAttribute("v-html") == "messagespan") {
                    div.innerHTML = messagespan
                }
            }
            // v-model
            var inputs = container.getElementsByTagName("input")
            for (var input of inputs) {
                if (input.getAttribute("v-model") == "message") {
                    input.setAttribute("value", message)
                }
            }
            // v-on
            var myInput = document.getElementById("myInput")
            myInput.onfocus = eval(myInput.getAttribute("v-on:focus"))
            myInput.onblur = eval(myInput.getAttribute("v-on:blur"))

            function focus(){
                myInput.setAttribute("value", "focus")
            }

            function blur() {
                myInput.setAttribute("value", "blur")
            }
            // v-for
            var liContents = [
                "jack",
                "rose",
                "james",
                "wade",
                "jordan"
            ]

            var liElementList = []
            for(var content of liContents) {
                var li = document.createElement("li")
                li.innerHTML = `<label><input type="checkbox"/><span>${content}</span></label>`
                liElementList.push(li)
            }
            var ul = container.getElementsByTagName("ul")[0]
            for (var liEle of liElementList) {
                ul.appendChild(liEle)
            }
        </script>
    </body>
</html>
```
点击[此处](https://jsfiddle.net/VioletJack/cpsrzbx5/5/)看效果。最后结果如图：
![显示结果](https://upload-images.jianshu.io/upload_images/1987062-8c87d7451a8f3abf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单实现了 Vue.js 指令的这些功能，其实在 Vue.js 源码中也是用了这些 dom 操作的 api 来做的。
更多 Vue 源码中的 DOM 操作可以看下我的 [《Vue.js 源码学习六 —— VNode虚拟DOM学习》](https://www.jianshu.com/p/9db8eb16d76f)这篇文章中。

# 最后

无论什么框架，其实都是万变不离其宗。最终都是用最基础的 API 来实现的各种功能。所以学好基础知识是非常重要的~
PS：还是 MDN 靠谱，w3school 的资料虽然也挺多，但是感觉不是很靠谱……以后查资料尽量去 MDN 英文网站去查（中文网站翻译有些问题）。