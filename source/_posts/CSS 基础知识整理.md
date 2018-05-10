---
title: CSS 基础知识整理
date: 2018-03-28
---

> 对于 CSS，由于过度依赖美工导致 CSS 很弱，所以这次好好学习 CSS相关东西。

# 盒子模型

盒子模型是由 margin、border、padding 和 content 属性组成的。

* margin 就是 border 外的透明边距区域。如 `margin: 10px;`。
* border 就是边框属性，可以定义边框的宽度、样式和颜色。如 `border: 5px solid red`。
* padding 就是 border 内的透明填充边距。如 `padding: 10px;`。
* content 也就是容器内要显示内容的区域。

盒子模型唯一要注意的就是边框的界定问题。IE 的 width 和 height 包括了 border 和 padding 的宽度，而 w3c （其他浏览器）的 wdith 和 height 只作用域 content，与 padding、padding 都区分出来的。
即在 IE 中，`content.width + padding.width * 2 + border.width * 2 = width`。
w3c 中 `content.width = width`。

```css
div {
  margin: 10px;
  border: 2px solid red;
  padding: 5px;
  height: 30px;
  width: 60px;
}
```

![chrome中的盒子模型](https://upload-images.jianshu.io/upload_images/1987062-41823df9d5d2ab83.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# overflow

* visible 超出边框显示
* hidden 隐藏超出文本
* scroll 换行滚动
* auto 自动换行
* inherit 继承父级CSS的属性

关于 scroll 和 auto 的区别，这篇[文章](https://www.jianshu.com/p/1ac2933891cd)中讲的很详细。

# display

* none 隐藏，不占空间
* block 元素间换行，带有换行符
* inline 默认，元素间不换行。宽度满了之后换行显示，流式布局显示。
* inline-block 和 inline 效果相同，流式布局显示。于 inline 不同之处在于 inline 不可设置宽高，inline-block 可以设置宽高。
* list-item 作为列表元素，换行显示。
* table 块级元素表格，后面带有回车。与 block 和 list-item 不同之处于宽度不是撑满整个页面。
* flex 将当前容器元素变为 flex 弹性盒模型。

其他 display 还有许多 table-*** 属性，感觉不常用暂且忽略了。

# position

* absolute 对非 static 父元素的绝对定位，脱离流。
* fixed 对于浏览器窗口的绝对定位，悬浮，脱离流。
* relative 相对父元素定位，在流中。
* static 默认值，没有定位，在流中。

# left、right、top、bottom

这四个属性定义了**定位元素相应外边距边界**与**其包含块相应边界**之间的偏移。如 `left: 20;px` 就是子元素相对于父元素向右偏移移 20px。
如果元素定义为 absolute，不设置 width 和 height，而设置上下左右四个属性那么就会根据父级元素容器来定义元素的宽高。
```css
.child {
  position: absolute;
  top: 10px;
  left: 15px;
  bottom: 20px;
  right: 25px;
  background-color: green;
}
```
如果同时设置了 `width` `height` `top` `left` `right` `bottom`，那么就取 `width` `height` `top` `left` 属性。
优先级而言：宽高优先级最高，`top` `left` 其次，`right` `bottom` 最后。

# 最后

暂时就写这么多吧，把常见的没搞懂的 CSS  给过了一遍，要想更好的掌握 CSS，必须多多动手实践才可以。
如有其他常用的 CSS 属性将会继续更新~