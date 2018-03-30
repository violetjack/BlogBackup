---
title: element 源码学习四 —— color-picker 源码学习
date: 2018-03-31
tag: "element源码学习"
---

> 在 element ui 中最让我好奇的组件之一就是 color-picker 着色器组件。这里还是通过几个问题来学习一下如何实现着色器的。

# 源码地址

在前几篇博客中说起过 element 组件都位于 `package` 目录下，那么本次学习的颜色选择器就是在 `package/color-picker` 目录中。
简单说下目录结构：
![目录结构](https://upload-images.jianshu.io/upload_images/1987062-34971b1038d38db9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **src** 源码文件夹
  * **components** 组件文件夹
    * **alpha-slider.vue** 透明度选择器
    * **hue-slider.vue** 色调选择器
    * **picker-dropdown.vue** 下拉界面（几个选择器的组合）
    * **sv-panel** 颜色选择器
  * **color.js** 颜色处理逻辑
  * **draggable.js** 选择器拖动效果逻辑
  * **main.vue** color-picker 的整体界面实现。
* **cooking.conf.js** [cooking](http://cookingjs.github.io/zh-cn/index.html) 配置
* **index.js** index文件，用于导出组件
* **package.json** 组件信息配置文件

下面通过问答解决问题的方式来学习 color-picker 组件。

# 回答几个源码问题

## 整体组件的结构是怎样的？

从整体结构来看，color-picker 的结构其实是多个组件的组合而成的。
* **显示颜色结果的 span** 和**选择颜色的下拉框**组成整体的 color-picker 组件；
* 其中**下拉框**由以下组件组合而成；
  * 3个颜色选择器
  * 1个input
  * 1个清空button
  * 1个确定button

![组件结构](https://upload-images.jianshu.io/upload_images/1987062-f84f7d12052e86d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![结构图](https://upload-images.jianshu.io/upload_images/1987062-c9e4c7f3c5b7a2aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 选择器的背景颜色变化是如何实现的？

3 个颜色选择器都是由 CSS3 的线性渐变效果 [linear-gradient()](https://developer.mozilla.org/en-US/docs/Web/CSS/linear-gradient) 来实现的。下面是简化版~
```html
    <style>
        .div01 {
            width: 27px;
            height: 350px;
            background: linear-gradient(to bottom, red 0, #ff0 17%, #0f0 33%, #0ff 50%, #00f 67%, #f0f 83%, red 100%);
        }

        .bg-white {
            width: 450px;
            height: 350px;
            position: absolute;
            background: linear-gradient(to right, #fff, rgba(255, 255, 255, 0));
        }

        .bg-black {
            width: 450px;
            height: 350px;
            position: absolute;
            background: linear-gradient(to top, #000, transparent);
        }

        .div02 {
            width: 450px;
            height: 350px;
            position: relative;
            background: rgb(213, 0, 255);
        }

        .div03 {
            height: 27px;
            width: 450px;
            background: linear-gradient(to right, rgba(213, 0, 255, 0) 0%, rgb(213, 0, 255) 100%);
        }
    </style>

    <div class="div02">
        <div class="bg-white"></div>
        <div class="bg-black"></div>
    </div>
    <div class="div01"></div>
    <div class="div03"></div>
```

最终结果如图所示：![显示结果](https://upload-images.jianshu.io/upload_images/1987062-0318237c94811e07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原来看似复杂的颜色选择器知识用了几个渐变就组合出来了，CSS 真的很强大！

## 如何计算并获取选中的色值？

颜色结果的计算逻辑都在 color.js 中了，来简单看下代码。
```js
// hsv 转 hsl
const hsv2hsl = function(hue, sat, val) {};

// 是否为 1.0
const isOnePointZero = function(n) {};

// 是否为百分比
const isPercentage = function(n) {};

// Take input from [0, n] and return it as [0, 1]
const bound01 = function(value, max) {};

// 十进制转十六进制
const INT_HEX_MAP = { 10: 'A', 11: 'B', 12: 'C', 13: 'D', 14: 'E', 15: 'F' };

// 转为十六进制颜色值
const toHex = function({ r, g, b }) {};

// 十六进制转十进制
const HEX_INT_MAP = { A: 10, B: 11, C: 12, D: 13, E: 14, F: 15 };

// 解析十六进制
const parseHexChannel = function(hex) {};

// hsl 转 hsv
const hsl2hsv = function(hue, sat, light) {};

// rgb 转 hsv
const rgb2hsv = function(r, g, b) {};


// hsv 转 rgb
const hsv2rgb = function(h, s, v) {};

export default class Color {
  constructor(options) {
    this._hue = 0;
    this._saturation = 100;
    this._value = 100;
    this._alpha = 100;

    this.enableAlpha = false;
    this.format = 'hex';
    this.value = '';

    options = options || {};

    for (let option in options) {
      if (options.hasOwnProperty(option)) {
        this[option] = options[option];
      }
    }

    this.doOnChange();
  }
  // 设置属性值
  set(prop, value) {
    if (arguments.length === 1 && typeof prop === 'object') {
      for (let p in prop) {
        if (prop.hasOwnProperty(p)) {
          this.set(p, prop[p]);
        }
      }

      return;
    }

    this['_' + prop] = value;
    this.doOnChange();
  }
  // 获取属性值 _hue
  get(prop) {
    return this['_' + prop];
  }
  // 颜色值转为 rgb 返回
  toRgb() {
    return hsv2rgb(this._hue, this._saturation, this._value);
  }
  // 格式化传入的值
  fromString(value) {
    if (!value) {
      this._hue = 0;
      this._saturation = 100;
      this._value = 100;

      this.doOnChange();
      return;
    }
    // 定义计算出结果后：赋值、改变。
    const fromHSV = (h, s, v) => {
      this._hue = Math.max(0, Math.min(360, h));
      this._saturation = Math.max(0, Math.min(100, s));
      this._value = Math.max(0, Math.min(100, v));

      this.doOnChange();
    };

    /* 颜色变化逻辑，最后都会转为 HSV 三个值执行 fromHSV 方法 */
  }

  // 更具计算结果定义当前颜色值 value
  doOnChange() {
    const { _hue, _saturation, _value, _alpha, format } = this;

    if (this.enableAlpha) {
      switch (format) {
        case 'hsl':
          const hsl = hsv2hsl(_hue, _saturation / 100, _value / 100);
          this.value = `hsla(${ _hue }, ${ Math.round(hsl[1] * 100) }%, ${ Math.round(hsl[2] * 100) }%, ${ _alpha / 100})`;
          break;
        case 'hsv':
          this.value = `hsva(${ _hue }, ${ Math.round(_saturation) }%, ${ Math.round(_value) }%, ${ _alpha / 100})`;
          break;
        default:
          const { r, g, b } = hsv2rgb(_hue, _saturation, _value);
          this.value = `rgba(${r}, ${g}, ${b}, ${ _alpha / 100 })`;
      }
    } else {
      switch (format) {
        case 'hsl':
          const hsl = hsv2hsl(_hue, _saturation / 100, _value / 100);
          this.value = `hsl(${ _hue }, ${ Math.round(hsl[1] * 100) }%, ${ Math.round(hsl[2] * 100) }%)`;
          break;
        case 'hsv':
          this.value = `hsv(${ _hue }, ${ Math.round(_saturation) }%, ${ Math.round(_value) }%)`;
          break;
        case 'rgb':
          const { r, g, b } = hsv2rgb(_hue, _saturation, _value);
          this.value = `rgb(${r}, ${g}, ${b})`;
          break;
        default:
          this.value = toHex(hsv2rgb(_hue, _saturation, _value));
      }
    }
  }
};
```
其中，将工具方法和计算颜色的具体方法隐藏了，只看具体逻辑。
其实 color.js 主要是定义了一个 Color 类，简单说下其中一些方法的作用：
* set 用于设置 Color 中的变量。
* get 用于获取 `_hue` `_saturation` `_value` `_alpha` 这四个值。
* toRgb 方法将当前颜色的值（除了透明度）以 RGB 的形式返回。
* fromString 方法将传入的颜色值解析成 HSV 格式，并赋值给 `_hue` `_saturation` `_value` 和 `_alpha`。
* doOnChange 方法将会计算颜色值组成字符串传给 `value`。

至此，Color 的大致功能就清晰了：**解析传入的颜色值为 HSVA 格式分别表示为 `_hue` `_saturation` `_value` 和 `_alpha`，并且组合成颜色字符串传给 `value`。**
现在需要把获取到的颜色值传给显示结果的 span，那么就从 `main.vue` 的中显示颜色结果的 `<span>` 标签开始看起。
```html
<span class="el-color-picker__color-inner" :style="{ backgroundColor: displayedColor }"></span>
```
背景色调用了 displayedColor 这个 computed 属性：
```js
    computed: {
      displayedColor() {
        if (!this.value && !this.showPanelColor) {
          return 'transparent';
        } else {
          const { r, g, b } = this.color.toRgb();
          return this.showAlpha
            ? `rgba(${ r }, ${ g }, ${ b }, ${ this.color.get('alpha') / 100 })`
            : `rgb(${ r }, ${ g }, ${ b })`;
        }
      },
  }
```
这里的 `this.value` 是 props 中传入的属性。如果没有传入 `value` 并且没有选择过颜色，那么显示透明色；
而 `this.color` 是 Color 类的实例化对象：
```js
const color = new Color({
   enableAlpha: this.showAlpha,
   format: this.colorFormat
 });
```
所以，就调用了我们上面所说的 `toRgb` 方法，最后返回颜色结果。
至此，实现了颜色的计算、获取和显示。

## 颜色选择器如何获取和修改颜色值？

在下拉菜单中 `hue-slider` 组件获取色调（哪种颜色）、`sv-panel` 获取具体的颜色值、`alpha-silder` 获取透明度。
这三个组件通过 props 获取父级组件传递的的 `color` 对象来显示颜色。如果颜色选择器的选择块移动后，通过修改 `color` 值来实现颜色的修改。

## 颜色选择器的选择块如何实现

选择颜色的过程其实就是选择器位移发生变化的过程。下面是作者参照 element 做的一个在有限范围内任意移动选择器的 demo：
```html
    <style>
        #container {
            width: 500px;
            height: 500px;
            position: relative;
            border: 1px solid black;
        }

        .drag {
            height: 4px;
            width: 4px;
            position: absolute;
            border-radius: 50%;
            border: 1px solid red;
            cursor: pointer;
        }
    </style>

    <div id="app">
        <div id="container" ref="container">
            <div class="drag" 
            :style="{
                top: cursorTop + 'px',
                left: cursorLeft + 'px'
            }"></div>
        </div>

    </div>

    <script>
        new Vue({
            el: "#app",
            data: {
                cursorLeft: 0,
                cursorTop: 0,
            },
            mounted() {
                draggable(this.$el, {
                    drag: (event) => {
                        this.handleDrag(event);
                    },
                    end: (event) => {
                        this.handleDrag(event);
                    }
                });

                this.update();
            },
            methods: {
                handleDrag(event) {
                    const container = this.$refs.container
                    const el = this.$el;
                    const rect = container.getBoundingClientRect();

                    let left = event.clientX - rect.left;
                    let top = event.clientY - rect.top;
                    left = Math.max(0, left);
                    left = Math.min(left, rect.width - 6);

                    top = Math.max(0, top);
                    top = Math.min(top, rect.height - 6);

                    this.cursorLeft = left;
                    this.cursorTop = top;
                }
            }
        })

        let isDragging = false;

        function draggable(element, options) {
            if (Vue.prototype.$isServer) return;
            const moveFn = function (event) {
                if (options.drag) {
                    options.drag(event);
                }
            };
            const upFn = function (event) {
                document.removeEventListener('mousemove', moveFn);
                document.removeEventListener('mouseup', upFn);
                document.onselectstart = null;
                document.ondragstart = null;

                isDragging = false;

                if (options.end) {
                    options.end(event);
                }
            };
            element.addEventListener('mousedown', function (event) {
                if (isDragging) return;
                document.onselectstart = function () { return false; };
                document.ondragstart = function () { return false; };

                document.addEventListener('mousemove', moveFn);
                document.addEventListener('mouseup', upFn);
                isDragging = true;

                if (options.start) {
                    options.start(event);
                }
            });
        }
    </script>
```
好吧，我知道代码太长了，要看效果请移步[此处](https://jsfiddle.net/VioletJack/ezttuvmf/1/)。
选择器的逻辑如下：

* 根据 props 传入的颜色值初次计算选择器的位置。
* 拖动选择器，根据选择器位置、已知的 color 属性计算当前选择器位置的颜色结果。

也就是说做一个选择器需要的就是**一个可拖动的选择器**和**一套计算颜色的算法逻辑**。比如在 `sv-silder` 中的算法逻辑如下：
```js
// 计算 cursor 位置
this.cursorLeft = saturation * width / 100;
this.cursorTop = (100 - value) * height / 100;
// 计算颜色
this.color.set({
  saturation: left / rect.width * 100,
 value: 100 - top / rect.height * 100
});
```
其他两个选择器原理也是类似的~

# 最后

至此，我对 color-picker 的一些疑惑都解开了，也写了一些 demo 来玩玩。对该组件有了大致的理解了~不得不感叹作者对于 CSS 和 Vue 的掌握真的非常熟练。学到了不少东西，感谢开源社区给我们提供了那么多好东西给我们使用和学习~
再下一篇文章中我想探索下其他一些有趣的 element 组件，敬请期待！