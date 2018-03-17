---
title: element 源码学习（番外篇） —— SASS五分钟快速入门
date: 2018-03-14
tag: "element源码学习"
---

> 这算是 element 源码学习的番外篇，因为 element 中使用了大量 sass 来写样式。而 UI 框架的核心其实就是样式。所以，抽空把 sass 学了一遍，写了些小 demo 实践，总结成此文。

# SASS 安装和调试

简单说下 sass 如何安装和编译调试。
参照[官网](http://sass-lang.com/install)，需要使用 gem 来安装 sass。如果是windows用户没有 gem 需要先安装 [Ruby](https://rubyinstaller.org/)
```shell
$ gem install sass
```
如果有权限问题，需要加上 `sudo` 。
```shell
$ sudo gem install sass
```
最后，通过查询 sass 版本号验证是否安装成功。
```shell
$ sass -v
```
编译命令很简单，在项目目录下编译选中 `.scss`、`.sass` 文件即可。
```shell
$ sass hello.scss hello.css
```
如果是学习 sass 这一个命令足矣，其他命令可参考[sass 编译](https://www.w3cplus.com/sassguide/compile.html)

# 语法简述

下面我用自己对 sass 语法的理解，配合上 demo 快速过一遍 sass 基础语法。

## sass 文件和 scss 文件区别

两者其实都是 sass 可以识别的文件，唯一不同点是 `.sass` 文件不使用大括号和分号。如下~
```sass
// .scss
$default-color: #FFAACC;
.selected {
    color: $default-color;
}

// .sass
$default-color: #FFAACC
.selected 
    color: $default-color

// .css
.selected {
  color: #FFAACC; 
}
```
官方文档推荐使用 `.scs`s 文件类型写法，避免 `.sass` 文件的严格格式要求报错。

## 变量

通过 `$` 符号来定义 sass 变量，变量在样式内外都可定义，用于各个样式中。定义的变量不会在编译后的 CSS 文件中显示。
```sass
$default-color: #FFAACC;
$border-color: #AAFFCC;
$default-border:  1px solid $border-color;
$extra-color: #BBDD00;

.selected {
    $scoped-width: 60px;
    width: $scoped-width;
    color: $default-color;
    border: $default-border;
}
```
编译结果：
```css
.selected {
  width: 60px;
  color: #FFAACC;
  border: 1px solid #AAFFCC; }
```
另外注意的一个点是在定义变量时使用的 `-` 和 `_` 的效果是一样的。即 `$border-color` 和 `$border_color` 指向的是同一个变量。

## 嵌套

为了避免一些代码的重复，引入了代码的嵌套。看一个例子：
```sass
.selected {
    color: #FFAA22;
    h1 {
        color: #FFDD77
    }
    div {
        width: 50px;
        height: 20px;
        span {
            color: #012DD6;
        }
    }
    &:hover {
        color: #FAFAFA;
    }
}
```
得到的结果如下：
```css
.selected {
  color: #FFAA22; }
  .selected h1 {
    color: #FFDD77; }
  .selected div {
    width: 50px;
    height: 20px; }
    .selected div span {
      color: #012DD6; }
  .selected:hover {
    color: #FAFAFA; }
```
从中可以看到，嵌套可以将需要重复写选择器的过程嵌套到一个表达式中了。

### 父选择器标识符 &

从上面的例子中看到有这么一段 `&:hover` 而在编译结果中得到的结果是 `.selected:hover` ，其实 `&` 标识符就代表了父级选择器。就这么简单，不理解的时候把父级选择题替换 `&` 理解下就简单了。

### 嵌套css

sass 中的嵌套是可以多个样式同时嵌套的。 用法见demo。
```sass
.container .content {
    h1, h2, h3 {margin-bottom: .8em}
}
```
编译结果
```css
.container .content h1, .container .content h2, .container .content h3 {
  margin-bottom: .8em; }
```

### 子组合选择器和同层选择器
关于 `>`、`+` 和 `~` 这三个选择器，是 CSS3 中就有的。在 SASS 中同样适用。
简单说下三个选择器用途（参考[ CSS 选择器](http://www.w3school.com.cn/cssref/css_selectors.asp)）：

* `div>p` 选取父元素是 <div> 元素的每个 <p> 元素
* `div+p` 选择 <div> 元素之后紧跟的每个 <p> 元素
* `p~ul` 选择前面有 <p> 元素的每个 <ul> 元素。

看个官网的例子：
```sass
article {
    ~ article { border-top: 1px dashed #ccc }
    > section { background: #eee }
    dl > {
      dt { color: #333 }
      dd { color: #555 }
    }
    nav + & { margin-top: 0 }
}
```
动手编译出的结果如下：
```css
article ~ article {
  border-top: 1px dashed #ccc; }
article > section {
  background: #eee; }
article dl > dt {
  color: #333; }
article dl > dd {
  color: #555; }
nav + article {
  margin-top: 0; }
```

### 嵌套属性

属性嵌套说白了就是把 `margin-bottom` 这类有 `-` 符号隔开的属性拆分开来便于查看和编写。
```sass
nav {
    border: {
    style: solid;
    width: 1px;
    color: #ccc;
    }
  }
```
这里把 `border-style` 等属性进行了拆分，结果如下：
```css
nav {
  border-style: solid;
  border-width: 1px;
  border-color: #ccc; }
```

## 导入

sass 提供了 sass 文件导入功能。可以做一些基础样式的复用。用法很简单：
```sass
// var.scss
$default-color: #AABBCC;

.focused {
    color: red;
    margin: 5px;
}

// h1.scss
h1 {
    color: #BBDDFF;
    margin: 10px;
}

// demo.scss
@import "var01";

$default-color: #FFAADD !default;

.selected {
    color: $default-color;
    @import "h1";
}
```
以上代码中将 `var.scss` 和 `h1.scss` 两个文件导入到了 `demo.scss` 中，最终生成结果如下：
```css
.focused {
  color: red;
  margin: 5px; }

.selected {
  color: #AABBCC; }
  .selected h1 {
    color: #BBDDFF;
    margin: 10px; }
```
从结果来说说导入的几个注意点：

* 导入文件使用 `@import` 表达式来导入，导入可以是外部导入也可是嵌套导入。上面例子中 `var` 是外部导入，而 `h1` 是嵌套导入的。
* `$default-color: #FFAADD !default;` 中的 `!default` 是定义变量默认值的方式，如果导入文件中有同样的值优先使用导入的值，如果导入文件中没有这个值，使用默认值。
* sass 中的 `@import` 与 css 中的 `@import` 不同，sass 中是编译的时候就直接导入生成 css 文件了，而 css 中，只有执行到 `@import` 时，浏览器才会去下载 css 文件，会导致页面加载变慢。

## 静默注释

就是在 sass 是否保留注释内容的语法。保留注释的方式为：**在CSS语法允许的地方，以 `/*...*/` 的方式写注释就能在生成的 css 文件中看到。**再来看一个例子：
```sass
// 不显示
/* 显示 */

.selected {
    //不显示
    /* 显示 */
    color: #FFAADD; // 不显示
    margin:/* 不显示 */ 10px; /* 显示 */
    /* 显示 */border: 1px dashed /* 不显示 */ #ccc;
}

// 不显示
/* 显示 */
```
编译结果正如注释所预测的。

## 混合器

混合器在 element 的样式表中用的非常多，是个很强大的功能。混合器以 `@mixin` 来导出混合内容，使用 `@include` 来导入混合内容。

### 基本用法

我的理解是：`@mixin` 类似定义变量一样定义个混合器（编译的时候不显示 `@mixin` 的内容），`@include` 获取混合器来替换 `@include xxxx` 的这行内容。
```sass
// mixin.scss
@mixin rounded-corners {
    -moz-border-radius: 5px;
    -webkit-border-radius: 5px;
    border-radius: 5px;
    div {
        width: 50px;
        height: 20px;
    }
}

.name {
    span {
        color: #FFAADD;
    }
}

// include.scss
@import "mixin";

.notice {
    background-color: green;
    border: 2px solid #00aa00;
    @include rounded-corners;
}
```
这里在官方 demo 上加了点代码验证问题。得到结果如下：
```css
.name span {
  color: #FFAADD; }

.notice {
  background-color: green;
  border: 2px solid #00aa00;
  -moz-border-radius: 5px;
  -webkit-border-radius: 5px;
  border-radius: 5px; }
  .notice div {
    width: 50px;
    height: 20px; }
```
这个demo验证了：1. `@mixin` 是一个类似变量的内容。在 `@import` 导入的内容中只显示了 `.name` 样式。  2. `@include` 会替换 `@include xxx` 这段代码。就算 `@mixin` 中有各种写法都会应用到 `@include`中，如嵌套 CSS。

### 混合器传参
混合器可以接收 `@include` 表达式传递的参数。而且，参数可以设置默认值。
```sass
// mixin.scss
@mixin link-colors($normal: white, $hover: white, $visited: white) {
    color: $normal;
    &:hover { color: $hover; }
    &:visited { color: $visited; }
}

// include.scss
@import "mixin";

// 写法一，按照默认顺序传递参数
a {
    @include link-colors(blue, red, green);
}

// 写法二，按照参数名传递参数
b {
    @include link-colors(
      $normal: blue,
      $visited: green,
      $hover: red
  );
}
```
编译结果为:
```css
a {
  color: blue; }
  a:hover {
    color: red; }
  a:visited {
    color: green; }

b {
  color: blue; }
  b:hover {
    color: red; }
  b:visited {
    color: green; }
```
如果说 `@include` 中不传递参数 `@include link-colors();` ，那么生成结果的 color 都为默认值 white。

### element 中的混合

在 element 源码中用了不少混合，有一种写法 sass 的快速入门中没有提到。就找一个简单的 el-card 样式来学习下来：
```sass
@import "mixins/mixins";
@import "common/var";

@include b(card) {
  border-radius: $--card-border-radius;
  border: 1px solid $--card-border-color;
  background-color: $--color-white;
  overflow: hidden;
  box-shadow: $--box-shadow-light;
  color: $--color-text-primary;

  @include e(header) {
    padding: #{$--card-padding - 2 $--card-padding};
    border-bottom: 1px solid $--card-border-color;
    box-sizing: border-box;
  }

  @include e(body) {
    padding: $--card-padding;
  }
}
```
从中可以看到所有的属性都是使用了 `@include` 方式进行混合的。最终生成的 CSS 文件如下：
```css
.el-card {
  border-radius: 4px;
  border: 1px solid #ebeef5;
  background-color: #fff;
  overflow: hidden;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  color: #303133; }
  .el-card__header {
    padding: 18px 20px;
    border-bottom: 1px solid #ebeef5;
    box-sizing: border-box; }
  .el-card__body {
    padding: 20px; }
```
先看下导入，其中 `var` 文件是项目样式变量统一保存的地方；`mixin` 文件用于混合；再加上几个 `@include` 表达式，答案一定是在 `mixin` 中的。我们就直接找到 `@mixin b` 和 `@mixin e` 两个混合项。
```sass
@mixin b($block) {
  $B: $namespace+'-'+$block !global; // 定义 B 变量：变量名 el-card

  .#{$B} {
    @content;
  }
}

@mixin e($element) {
  $E: $element !global;
  $selector: &;
  $currentSelector: "";
  @each $unit in $element {
    $currentSelector: #{$currentSelector + "." + $B + $element-separator + $unit + ","};
  }

  @if hitAllSpecialNestRule($selector) {
    @at-root {
      #{$selector} {
        #{$currentSelector} {
          @content;
        }
      }
    }
  } @else {
    @at-root {
      #{$currentSelector} {
        @content;
      }
    }
  }
}
```
发现有许多 sass 快速入门中没有提到过的语法： `@if`，`@else` 等等，这里查阅具体文档列出其功能：

* @if @else 这两者和任何编程语言的 if ... else ... 的用法是一样的，条件判断。if 中条件为 true 进入逻辑，否则使用 else 逻辑。
* @at-root [@at-root](https://www.sasscss.com/docs/#at-root) 指令导致一个或多个规则被限定输出在文档的根层级上，而不是被嵌套在其父选择器下。
* @content 样式内容块可以传递到混入（mixin）包含样式的位置。样式内容块将出现在混入内的任何 @content 指令的位置。这使得可以定义抽象 关联到选择器和指令的解析。
* @each in 类似js用法，遍历列表获取每个value值。
* `#{...}` 是[插值语法](https://www.sasscss.com/docs/#interpolation_)，用于在选择器和属性名中使用 SassScript 变量，所以 `.#{$B}` 表达式，如果 `$B` 的值为 hello-world，那么表达式结果等于 `.hello-world`

其实看完这些用法，上面的代码就很好理解了。具体关于 element 样式学习的细节将在下篇博客中详细学习。

## 继承

个人感觉继承就是几个样式类写在一起。而且，继承是可以嵌套的。
```sass
.error {
    border: 1px red;
    background-color: #fdd;
}
  
.seriousError {
    @extend .error;
    border-width: 3px;
}

.error02 {
    @extend .seriousError;
    margin: 5px;
}
```
编译结果为：
```css
.error, .seriousError, .error02 {
  border: 1px red;
  background-color: #fdd; }

.seriousError, .error02 {
  border-width: 3px; }

.error02 {
  margin: 5px; }
```
下面引用下继承的注意事项：

> * 跟混合器相比，继承生成的 css 代码相对更少。因为继承仅仅是重复选择器，而不会重复属性，所以使用继承往往比混合器生成的 css 体积更小。如果你非常关心你站点的速度，请牢记这一点。
>* 继承遵从 css 层叠的规则。当两个不同的 css 规则应用到同一个 html 元素上时，并且这两个不同的 css 规则对同一属性的修饰存在不同的值，css 层叠规则会决定应用哪个样式。相当直观：通常权重更高的选择器胜出，如果权重相同，定义在后边的规则胜出。

所以，其实继承相比混合更简单。继承只是选择器的重复，而混合是用一段代码替换标签 `@include` 标签。

# 最后

由于 element 项目中使用了大量的 sass 样式，所以想了解 element 必须对 sass 有一定了解。本文简单解决了 sass 是什么？基础用法怎么用？两个问题。更加深入的 sass 语法涉及的不多，算是快速入门博客啦。
在了解了 sass，能够看懂 element 中的样式表后，就可以愉快的去学习 element 源码啦~