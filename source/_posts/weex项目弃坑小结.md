---
title: weex项目弃坑小结
date: 2018-02-06 16:20:52
---

> 由于`weex` 的不稳定性，所以中途放弃了 `weex` 方案。转而使用 Android原生开发。今天项目第一阶段开发结束，来总结一下 `weex` 的一些东西。

`weex` 相比于原生开发的好处

* 界面搭建速度快，特别是重复性界面。
* 数据处理方便，尤其是json解析方面。

坏处么

* 用的人太少，所以网上资料也很少。
* 坑还是比较多的，要玩的溜要踩上不少坑。
* 还是需要一些原生开发的知识才能够玩的溜。

说说我这个项目的一些东西吧~

## 页面跳转
第一个困扰我的就是页面跳转，我去SF上提问了[Weex的页面跳转方案的选择](https://segmentfault.com/q/1010000009999942)这个问题。

* `vue-router` 方案由于 `Vue`和 `weex` 的差异性，用法有所不同，好像需要使用注入 `minixs` 机制，挺麻烦的，所以放弃了……这个如果有需要我会再去研究一下的。
* 第二种方案就是每个页面都是单独的 `Activity`，各自嵌入一个 `weex` 页面。但是有个问题：`weex` 如何与当前 `Activity` 交互，比如我要从页面A跳转到页面B，我需要 `weex` 调用 `Activity` 的 `startActivity` 方法，这个交互方式我没有找到。不然倒是可以随心所欲的在原生开发和weex之间来回交互。暂时来说我只知道[通过globalEvent和Module扩展来实现两者的交互](http://blog.csdn.net/violetjack0808/article/details/74002599)
* 最后，还是选择了官方推荐的 `Navigator` 来作为页面跳转的方式。这种方式能够很快速的实现页面跳转。不用瞎折腾~

选定方案之后，就要解决几个问题了。多页面打包和跳转所需的 `Activity` 。
### 多页面打包
`navigator` 的跳转方式，需要获取打包好的 `weex` 的文件路径来进行页面跳转和显示，所以我们就需要多页 `weex` 文件代表每一个页面。
```
navigator.push({
  url: 'http://dotwe.org/raw/dist/519962541fcf6acd911986357ad9c2ed.js',
  animated: "true"
}, event => {
  modal.toast({ message: 'callback: ' + event })
})
```
需要配置 `Webpack` 来实现这一目的。我将每个页面的入口文件都放在 `./src/entrys`文件夹下，通过 `node` 的 `fs文件模块` 读取里面的入口文件，并将他们传给 `entry` 入口对象。最后将入口对象配置到 `webpack` 打包配置中。
```
var path = require('path')
var webpack = require('webpack')
var fs = require('fs')

var files = fs.readdirSync('./src/entrys')
var entry = {}
files.forEach(function (file) {
  var item = file.replace('.js', '')
  entry[item] = path.resolve('./src/entrys/' + file)
})

var bannerPlugin = new webpack.BannerPlugin(
  '// { "framework": "Vue" }\n', { raw: true }
)

function getBaseConfig() {
  return {
    entry: entry,
    output: {
      path: 'dist'
    },
    resolve: {
      alias: {
        '@': path.resolve('./src'),
        'views': path.resolve('./src/views'),
        'utils': path.resolve('./src/utils')
      }
    },
    module: {
      // ESLint配置
      preLoaders: [{
        test: /\.vue$/,
        loader: 'eslint',
        exclude: /node_modules/
      },
      {
        test: /\.js$/,
        loader: 'eslint',
        exclude: /node_modules/
      }
      ],
      // 如果注释掉以上这段将不产生ESLint检查
      loaders: [{
        test: /\.js$/,
        loader: 'babel',
        exclude: /node_modules/
      }, {
        test: /\.vue(\?[^?]+)?$/,
        loaders: []
      }]
    },
    vue: {
    },
    plugins: [bannerPlugin]
  }
}

var webConfig = getBaseConfig()
webConfig.output.filename = '[name].web.js'
webConfig.module.loaders[1].loaders.push('vue')

var weexConfig = getBaseConfig()
weexConfig.output.filename = '[name].weex.js'
weexConfig.module.loaders[1].loaders.push('weex')

module.exports = [webConfig, weexConfig]
```
每一个入口文件都会产生两个相应名称的文件，如 `sign.js` 的入口文件就会生成 `sign.weex.js` 和 `sign.web.js` 文件，这里我们只关注 `weex` 后缀的文件，这就是我们跳转页面所需的文件。具体项目结构请看[源代码](https://github.com/violetjack/MobileNurseWeex)
更多对于 `webpack` 的了解可以看[Vue.js学习系列四——Webpack学习实践](http://blog.csdn.net/violetjack0808/article/details/54915825)

### navigator跳转的Activity哪里来的？
在使用了 `navigator` 后一开始发现并没有页面跳转效果，而是报 `ActivityNotFoundException` 的错误。后来装了 `weex` 的 playground 之后发现可以跳转了，但是跳转过去的 `Activity` 有个 ActionBar 和一个性能调试的悬浮窗，是 playground 里面扫二维码显示的结果的那个 `Activity`。几经查阅后发现原来跳转的 `Activity` 是一个有着特殊 `intent-filter` 的 `Activity` 。
关于这个问题我写过一篇文章：[WEEX 使用navigator跳转Android系统出现ActivityNotFoundException报错](http://blog.csdn.net/violetjack0808/article/details/74390249)
解决方案是我把 Playground 里面的那个 `Activity` 移到了我的项目中来，并且去除了 `ActionBar` 和调试工具。然后卸载掉 `weex`  的 Playground，这样就能愉快的显示 `navigator` 跳转的 `Activity` 了。代码请看[WXPageActivity](https://github.com/violetjack/MobileNurseWeex/blob/master/android/app/src/main/java/com/weex/sample/activity/WXPageActivity.java)。

## 网络通信
网络通信上我用的是stream，这个很简单。说下两点：
第一，stream所用的url需要是UTF-8格式的，如果URL中有中文需要转一下，URL可以参照utf8这个文件。
第二，提交数据的时候需要添加头文件。我的项目中是这样~
```
      stream.fetch({
        method: StreamType,
        type: 'json',
        headers: {
          'Content-Type': 'application/json'
        },
        url: url,
        body: JSON.stringify(this.DataObj)
      }, res => {
        console.log(res)
        let json = eval('(' + res.data + ')')
        modal.alert({
          message: json.Message
        }, event => {

        })
      })
```

## 图片加载
图片加载需要在[ImageAdapter](https://github.com/violetjack/MobileNurseWeex/blob/master/android/app/src/main/java/com/weex/sample/ImageAdapter.java)中稍作处理，我这里用的是 `Picasso` 来显示图片的~
```
public class ImageAdapter implements IWXImgLoaderAdapter {
    
    @Override
    public void setImage(final String url, final ImageView view, WXImageQuality quality, WXImageStrategy strategy) {
        WXSDKManager.getInstance().postOnUiThread(new Runnable() {
            @Override
            public void run() {
                if (view == null || view.getLayoutParams() == null) {
                    return;
                }
                if (TextUtils.isEmpty(url)) {
                    view.setImageBitmap(null);
                    return;
                }
                String temp = url;
                if (url.startsWith("//")) {
                    temp = "http:" + url;
                }
                if (view.getLayoutParams().width <= 0 || view.getLayoutParams().height <= 0) {
                    return;
                }
                Picasso.with(WXEnvironment.getApplication())
                        .load(temp)
                        .into(view);
            }
        }, 0);
    }
}
```

### 页面跳转的数据传输
这一点上，我只想到了不太优雅的方式——使用 `storage` 来保存和读取。比如我需要将表单ID传递到下一个页面我是这么做的：
```
    toDetail(AssessID) {
      storage.setItem('AssessID', AssessID)
      navigator.push({
        url: ViewServer + 'GAD.weex.js',
        animated: 'true'
      }, event => {
        console.log('successful entry')
      })
    },
```
到第二个页面去获取数据：
```
    getData() {
      let that = this
      storage.getItem('AssessID', event => {
        let AssessID = event.data
        console.log('AssessID = ' + AssessID)
      })
    },
```
如果是比较多的数据，我会将数据以 `json` 字符串的形式保存，在需要的时候获取字符串并解析为 `json` 对象。

### 如何在返回上一页面时做一些操作？
我的解决方法是在Activity的onResume方法中发送一个消息，然后在weex端添加监听事件。
**Android端**
```
    @Override
    protected void onResume() {
        super.onResume();
        if (mInstance != null) {
            mInstance.onActivityResume();

            new Handler().postDelayed(new Runnable() {
                public void run() {
                    Map<String, Object> params = new HashMap<>();
                    mInstance.fireGlobalEventCallback("onResume", params);
                }
            }, 500);
        }
    }
```
**weex端**
```
	addListener() {
      globalEvent.addEventListener('onResume', e => {
        storage.getItem('PopCallback', event => {
          if (event.data === 'update level list') {
            this.loadData()
            storage.setItem('PopCallback', '')
          }
        })
      })
    }
```
由于加载 `weex` 有一些延时，`onResume` 往往会比 `weex` 加载快，所以我在 `onResume` 中添加了0.5秒延时。之后在 `weex` 中添加监听器监听 `onResume` 的生命周期，并监听返回的数据。

### 如何控制weex的Slider显示第几页
想实现Android的ViewPager效果，使用了weex提供的slider组件，具体坚决方案看[如何控制weex的Slider显示第几页。](https://segmentfault.com/q/1010000010728251)

### 其他

* 图片的存放好像只能是网络图片的URL，所以我将所有图片都放到服务器上让weex去访问。
* CSS使用flex布局来做，官网上有例子。也可参照[Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)。

# 最后
啰嗦一大堆，希望某些东西能够对他人有所帮助吧~
项目地址[在此](https://github.com/violetjack/MobileNurseWeex)，由于是公司项目，所以把服务器地址去掉了，后端数据获取不了。不过代码都是在的，可以进行参考。希望能对大家有所帮助~
最终不用weex的原因么，因为这玩意看似简单，但是想实现点复杂的、和原生交互的功能都得折腾好一会儿。关键还是资料太少，不靠谱，万一哪个地方报个奇怪的错误找不出问题、查不到资料、看不懂源码，那不就挂了嘛~所以，爱折腾玩玩可以，放项目中还是有待考虑。毕竟还是要求稳~


