---
title: weex 踩坑记（持续更新中……）
date: 2016-08-03
tag: "weex"
---

> 消失了一个月，努力为新项目倒腾 weex 中，记录一下遇到的问题。之后还会持续更新~

目前，我使用的 `weex` 都是在[集成Weex到Android](http://weex.apache.org/cn/guide/integrate-to-your-app.html)来做的，项目使用的是 `weex-toolkit` 生成的项目模板，代码发布使用webpack打包成js放到服务器上，Android端读取服务器上的js来实现weex项目的。

# 杂七杂八的一些知识点

* 屏幕宽度为 750，高度一直没查到，我用的是1300，刚好显示下。长度单位要么不写，要么就是 px，效果都一样。
```
.item {
  height: 1300;
  width: 750px;
}
```
* 布局只支持 盒子模型、flex布局、relative定位，其他一些CSS不太支持。
* CSS的 margin、padding、border 不支持缩写。像 `border:5px solid red;` 这样写是不行滴。
* 暂不支持像 [mint-ui](http://mint-ui.github.io/#!/zh-cn) 这类 Vue Web UI 组件库。
* 页面的跳转可以通过 [vue-router](https://router.vuejs.org/zh-cn/) 或者 weex 的 [navigator](http://weex.apache.org/cn/references/modules/navigator.html) 组件来实现，可以参考我提问的[Weex的页面跳转方案的选择](https://segmentfault.com/q/1010000009999942)。
* 在 Android 中，`navigator` 的 `push` 方法跳转的 `Activity` 界面是需要处理的，需要创建一个带有特殊 `<intent-filter>` 标签的 Activity。假如手机中没有带有该 `<intent-filter>` 的 `Activity` 就不会发生跳转，报 `ActivityNotFoundException` 错误。而如果有多个带有该 `<intent-filter>` 的`Activity`，Android 系统会让我们去进行选择。注：这个带 `<intent-filter>` 的Activity 是跨 APP 的。可以参照[WEEX 使用navigator跳转Android系统出现ActivityNotFoundException报错](http://blog.csdn.net/violetjack0808/article/details/74390249)
* `image` 必须设置宽高，否则不显示。也不能使用 `img` 来显示图片
* Android 手机中显示图片需要在 `ImageAdapter` 中进行处理，官网只提供了处理的位置（有注释），但未对图片进行处理。我使用了 [picasso](https://github.com/square/picasso) 来对图片进行显示。
* native 和 weex 的通信通过自定义module或者发送全局事件来完成。参考[Weex控制Android返回键解决方案](http://blog.csdn.net/violetjack0808/article/details/74002599)里面有 native 端和 weex 端交互的细节。
* weex 在Android手机中的调试：
  * 在 weex 中使用 console.log 方法来打 log，打开Android Studio，在 logcat中可以过滤关键词 jslog 来获取log数据。
  * 如果 weex 报错，可以在 logcat 中查找错误，一般错误都好几行，很好找。
  * 建议使用ESLint先过滤一些简单的语法错误，减少手机端的调试成本。
* weex 其他调试方式
  * 手机安装 [Playground](http://weex.apache.org/cn/playground.html)，运行 weex 项目，网页打开 `http://localhost:8080/` 扫描二维码进行调试。
  * 可以在[Playground网页端](http://dotwe.org/vue/025db54e37123ab5336a4b848397660f)进行代码调试，但感觉遇到有组件的项目不太好调试。
  * 在项目运行(`npm run serve`)后，直接打开 `http://localhost:8080/` 也能看到网页版本的项目，可以直接调试，不过一些设计Native端的组件用不了。
* weex中的标签只支持官方提供的[内部组件](http://weex.apache.org/cn/references/components/index.html)，因为那些是会被渲染成native界面的。
* `v-bind:class`只能使用数组语法
* `stream` 的 `url` 选项好像默认不支持中文，需要将中文转为 `UTF-8` 来传输。
* weex 的 css 只支持 class 选择器，并且只支持单个类的选择器，如`.item .item-content {}`是错误的~
* weex有点击特效的，参照[伪类](http://weex.apache.org/cn/references/common-style.html#伪类-v0-9-5)。
* 不支持 `display:none` 即不支持 `v-show`，需要使用 `v-if` 来实现显示和隐藏。
* 默认flex布局，要设置 `flex-direction`。我怀疑我的web端显示错误可能就是没有设置 `flex-direction`，移动端没有错误是因为默认 `flex-direction:column`
* 存储、网络等很多都是异步的，需要注意顺序
* `storage` 只能存储字符串，取值后再转为json

## 结尾
暂时整理这么多，之后还会有其他的东西我会持续更新的~
Android 端的 [demo](https://github.com/violetjack/WeexComponents) 我会放到我的 Github 上去，之后我会让我的 IOS 小伙伴给一版 IOS 版本的壳子，到时候直接写weex项目，Android端和IOS端只需要更改一下渲染的js文件路径就可以显示了。

## 参考文档
* [weex填坑注意事项](https://github.com/dreamochi/DayDayUp/issues/77)