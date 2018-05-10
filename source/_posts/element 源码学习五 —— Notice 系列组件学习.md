---
title: element 源码学习五 —— Notice 系列组件学习
date: 2018-04-03
---

> 消息提示行为是开发中非常常见的功能，Element 为我们提供了非常好用和美观的消息提示组件。这里就简单学习下 Notice 组件的 CSS 和代码逻辑。

# 简介

Notice 包括了五类组件：

* **Alert** 用于页面中展示重要的提示信息。
* **Loading** 加载数据时显示动效。
* **Message** 常用于主动操作后的反馈提示。与 Notification 的区别是后者更多用于系统级通知的被动提醒。
* **MessageBox** 模拟系统的消息提示框而实现的一套模态对话框组件，用于消息提示、确认消息和提交内容。
* **Notification** 悬浮出现在页面角落，显示全局的通知提醒消息。

本文中不同角度来学习这些组件（这些组件有很多相似性，所以一起学习啦~）。

# demo

下面是参照 element ui 写的一个小demo，尝试着了解下其中的 CSS
贴代码：
```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <title>My Notice</title>
    <style>
        .success-color {
            background-color: #f0f9eb;
            color: #67c23a;
        }

        .info-color {
            background-color: #f4f4f5;
            color: #909399;
        }

        .warning-color {
            background-color: #fdf6ec;
            color: #e6a23c;
        }

        .error-color {
            background-color: #fef0f0;
            color: #f56c6c;
        }


        .alert-container {
            position: relative;
            padding: 8px 16px;
            border-radius: 5px;
            opacity: 1;
            align-items: center;
            overflow: hidden;
            display: flex;
        }

        .success-text {
            font-size: 13px;
            line-height: 18px;
            padding: 0 8px;
            color: #67c23a;
        }

        .close {
            font-size: 13px;
            position: absolute;
            top: 12px;
            right: 15px;
            cursor: pointer;
        }

        .loading-background {
            position: fixed;
            z-index: 2000;
            background-color: rgba(26, 26, 26, 0.9);
            margin: 0;
            top: 0;
            right: 0;
            bottom: 0;
            left: 0;
            transition: opacity 0.3s;
            z-index: 2000;
        }

        .loading-div {
            top: 50%;
            position: absolute;
            margin-top: -20px;
            height: 40px;
            width: 100%;
            align-items: center;
            justify-content: center;
            display: flex;

        }

        .loading-text {
            color: #f0f9eb;
            font-size: 15px;
            overflow: hidden;
            z-index: 2001;
        }

        .el-message {
            min-width: 380px;
            box-sizing: border-box;
            border-radius: 4px;
            border-width: 1px;
            border-style: solid;
            border-color: #ebeef5;
            position: fixed;
            left: 50%;
            top: 20px;
            transform: translateX(-50%);
            background-color: #edf2fc;
            transition: opacity 0.3s, transform 0.4s;
            overflow: hidden;
            padding: 15px 15px 15px 20px;
            display: flex;
            align-items: center;
        }
    </style>
</head>

<body>
    <div id="app">
        <div class="alert-container success-color">
            <span class="success-text">成功提示的文案</span>
            <span class="close">X</span>
        </div>
        <button id="showLoading">显示加载中</button>
        <button id="showMessage">显示信息</button>
        <div class="el-message" id="message" style="display:none;">
            <p>这是一条消息</p>
        </div>
    </div>

    <script>
        var alertContainer = document.getElementsByClassName("alert-container")[0]
        var close = document.getElementsByClassName("close")[0]
        close.onclick = function () {
            alertContainer.style = "display:none;"
        }

        var app = document.getElementById("app")
        var bg = document.createElement("div")
        bg.className = "loading-background"
        var div = document.createElement("div")
        div.className = "loading-div"
        var text = document.createElement("span")
        text.className = "loading-text"
        text.textContent = "加载中……"
        div.appendChild(text)
        bg.appendChild(div)


        document.getElementById("showLoading").onclick = function () {
            if (document.getElementsByClassName("loading-background").length > 0) {
                return;
            }
            app.appendChild(bg)
            setTimeout(() => {
                if (document.getElementsByClassName("loading-background").length > 0) {
                    app.removeChild(bg)
                }
            }, 3000)
        }

        document.getElementById("showMessage").onclick = function () {
            document.getElementById("message").style = "display:block;"
            setTimeout(() => {
                document.getElementById("message").style = "display:none;"
            }, 3000)
        }
    </script>
</body>

</html>
```
代码的运行结果请看 **[->这里<-](https://jsfiddle.net/VioletJack/h28fnfds/2/)**！

以上代码实现了
* Alert 的样式
* loading 的简单版本 
* message 的无动画版本。

只实现这三个功能的原因是另外两个功能和扩展功能都是基于这个demo扩展的。
Alert 是一个静态的文本内容，只是外部包裹了带样式的容器。可能再多个隐藏按钮和消息图标。
Loading 和 MessageBox 其实的基本逻辑是插入或显示新的内容，并在显示完成后消失。需要注意的就是后面要添加一层遮罩阴影，遮罩如果非全屏使用 position:absolute 而全屏则使用 position:fixed 覆盖。另外就是注意 z-index 属性将组件放到视图最上层。
Message 和 Notification 其实就是文本内容、图标和按钮组合容器的现实和隐藏过程。它们的过渡动画使用的是 vue 的[进入/离开 & 列表过渡](https://cn.vuejs.org/v2/guide/transitions.html)来实现。

# 通过几个问题来看源码

其实从样式上，上面的 demo 已经实现了大致的样子了。下面来看看组件的一些逻辑。源码内容比较多，所以就以问答的方式有目的的来看源码。

## Alert 的界面实现？
Alert 由图标、文本内容、描述内容和关闭按钮组成：
```html
  <transition name="el-alert-fade">
    <div
      class="el-alert"
      :class="[typeClass, center ? 'is-center' : '']"
      v-show="visible"
      role="alert"
    >
      <!-- 图标 -->
      <i class="el-alert__icon" :class="[ iconClass, isBigIcon ]" v-if="showIcon"></i>
      <div class="el-alert__content">
        <!-- 标题 -->
        <span class="el-alert__title" :class="[ isBoldTitle ]" v-if="title">{{ title }}</span>
        <slot>
          <!-- 插槽，默认插入描述文本 -->
          <p class="el-alert__description" v-if="description">{{ description }}</p>
        </slot>
        <!-- 关闭图标按钮 -->
        <i class="el-alert__closebtn" :class="{ 'is-customed': closeText !== '', 'el-icon-close': closeText === '' }" v-show="closable" @click="close()">{{closeText}}</i>
      </div>
    </div>
  </transition>
```
组件很简单，注释上都写了~组件还做了 `slot` 插槽拓展，可以在 `<el-alert>` 标签内插入自定义内容。

## 如何显示纯文本、HTML 和 VNode

显示文本使用 `{{ text }}` 指令来显示内容。
显示HTML使用 `v-html` 指令来渲染显示。
```html
<!-- 显示文本 -->
<p v-if="!dangerouslyUseHTMLString" class="el-message__content">{{ message }}</p>
<!-- 显示HTML -->
<p v-else v-html="message" class="el-message__content"></p>
```
使用 vue 的 render 函数生成 VNode 对象传给组件作为组件 `slot` 插槽的默认显示结果。
```js
  if (isVNode(instance.message)) {
    instance.$slots.default = [instance.message];
    instance.message = null;
  }
```

## 组件的过渡动画怎么来的

使用 Notice 系列组件时，发现组件的显示和消失都是有过渡动画的。界面看着更加友好和舒服。实现方式就是使用了 vue 的[进入/离开 & 列表过渡](https://cn.vuejs.org/v2/guide/transitions.html)来实现效果。

## 界面的关闭使用的是什么方式？
对于 loading 和 message-box ，在没有界面时会在 body 最后添加组件内容，显示过后使用 `v-show="false" `(`display:none;`) 隐藏，随后就调用显示和隐藏界面。
```js
// 将 message-box 组件加入到 body 中
document.body.appendChild(instance.$el);
```
对于 Alert 由于一开始就显示，只是删除按钮，所以只需修改 `v-show` 属性隐藏即可。
对于 message 和 notification，这两个组件可以多次弹出，逐个关闭（自动或手动）。所以这两个组件是保存在一个数组中，然后进行渲染的，关闭某个组件就是一个将组件从组件数组中移除的过程。
```js
// id 组件id
// useOnClose 自定义关闭函数
Message.close = function(id, userOnClose) {
  for (let i = 0, len = instances.length; i < len; i++) {
    if (id === instances[i].id) {
      // 找到组件，执行自定义关闭函数并从数组中移除
      if (typeof userOnClose === 'function') {
        userOnClose(instances[i]);
      }
      instances.splice(i, 1);
      break;
    }
  }
};
```

## message-box 的回调函数实现

对于 message-box 有两种函数回调：callback 函数和 Promise 函数。
```js
  if (typeof Promise !== 'undefined') {
    return new Promise((resolve, reject) => {

      msgQueue.push({
        options: merge({}, defaults, MessageBox.defaults, options),
        callback: callback,
        resolve: resolve,
        reject: reject
      });

      showNextMsg();
    });
  } else {
    msgQueue.push({
      options: merge({}, defaults, MessageBox.defaults, options),
      callback: callback
    });

    showNextMsg();
  }
```
当然，到这一步只是将组件配置和回调函数组成对象，并没有执行。执行实在 `showNextMsg` 方法中。`showNextMsg` 方法用于组合当前组件 `options` 并写入到 DOM 中，然后显示组件。其中有这么一段关于回调的：
```js
      // 如果 currentMsg.options.callback 为 undefined
      if (options.callback === undefined) {
        instance.callback = defaultCallback;
      }
```
所以再看看 `defaultCallback` 函数对象：
```js
const defaultCallback = action => {
  if (currentMsg) {
    let callback = currentMsg.callback;
    if (typeof callback === 'function') {
      if (instance.showInput) {
        callback(instance.inputValue, action);
      } else {
        callback(action);
      }
    }
    if (currentMsg.resolve) {
      if (action === 'confirm') {
        if (instance.showInput) {
          currentMsg.resolve({ value: instance.inputValue, action });
        } else {
          currentMsg.resolve(action);
        }
      } else if (action === 'cancel' && currentMsg.reject) {
        currentMsg.reject(action);
      }
    }
  }
};
```
这里就可以看到回调函数和 Promise 的调用和传参过程。当组件“关闭”的时候执行 callback 方法回调：
```js
      doClose() {
        ……
        setTimeout(() => {
          if (this.action) this.callback(this.action, this);
        });
      },
```
至此实现了回调及其传参。

## 如何实现显示多个 Notification 向下偏移插入？

偏移量计算在 Notification 的构造方法中计算获得当前组件 `verticalOffset`。
```js
const Notification = function(options) {
  // 服务器渲染
  if (Vue.prototype.$isServer) return;
  options = options || {};
  const userOnClose = options.onClose; // 自定义关闭
  const id = 'notification_' + seed++; // 组件 id
  const position = options.position || 'top-right'; // 位置
  // 关闭事件
  options.onClose = function() {
    Notification.close(id, userOnClose);
  };
  // 组件实例
  instance = new NotificationConstructor({
    data: options
  });
  // vnode
  if (isVNode(options.message)) {
    instance.$slots.default = [options.message];
    options.message = 'REPLACED_BY_VNODE';
  }
  instance.id = id;
  instance.vm = instance.$mount();
  // 添加实例
  document.body.appendChild(instance.vm.$el);
  instance.vm.visible = true;
  instance.dom = instance.vm.$el;
  instance.dom.style.zIndex = PopupManager.nextZIndex();
  // 偏移量计算
  let verticalOffset = options.offset || 0;
  instances.filter(item => item.position === position).forEach(item => {
    verticalOffset += item.$el.offsetHeight + 16;
  });
  verticalOffset += 16;
  instance.verticalOffset = verticalOffset;
  // 传入数组
  instances.push(instance);
  return instance.vm;
};
```
偏移量 `verticalOffset` 在组件 `package\notification/src/main.vue` 中使用。
```html
    <div :style="positionStyle" ></div>
```
```js
    computed: {
      // 正则匹配 position 是 top 还是 bottom
      verticalProperty() {
        return /^top-/.test(this.position) ? 'top' : 'bottom';
      },
      // 最后返回的内联样式
      positionStyle() {
        return {
          [this.verticalProperty]: `${ this.verticalOffset }px`
        };
      }
    }
```
如果 `position` 是 `bottom-left` 偏移量 `verticalOffset` 为 20，那么返回的内联样式就是：
```css
{
  bottom: 20px;
}
```
至此实现了偏移的功能。

# 最后

这里简单学习了一下 Notice 系列组件的样式和逻辑。从中学到了：

* 用 CSS 设计样式和动态修改样式属性。
* 用了许多 DOM 的操作
* 使用已有轮子 —— 用到了 Vue 的 directive、 transition 和 render。
* 组件设计方面，学到了使用数组管理多个同类组件；使用 `<slot>` 标签给开发者预留拓展空间，使用构造函数的方法来拓展组件逻辑并定义一些快捷方法。

个人感觉学习一些成熟组件的源码能够学到不少东西，收获不少。