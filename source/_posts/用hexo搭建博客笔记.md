---
title: 用hexo搭建博客笔记
data: 2018-02-07
tag: "hexo"
---

> 一直想有个自己的 Github.io 博客，感觉逼格能够上升一大截。

很久之前就看到网上各种博客搭建的文章，但是从内心中总感觉好像是个很麻烦的事情。所以，一直没有动手去做。

昨天，趁着年前工作不忙，搭建了个博客，这里记录下过程。
其实，搭建hexo博客是非常简单的事情。

# 安装前提
Mac安装前提

* Xcode
* Node.js
* Git

这三个玩意儿对于我们开发者基本都是有的，没有装个就好。

# 创建博客的过程
简单的几条 bash 命令就好。
```
$ npm install -g hexo-cli
$ hexo init [文件夹名]
$ cd [文件夹名]
$ npm install
```
以上步骤就已经安装完毕了。

# 常用命令
```
// 新建文章 layout为模板，title为文章名
$ hexo new [layout] <title>
// 启动本地服务器看hexo博客，地址为 `http://localhost:4000/`
$ hexo server
// 生成静态文件
$ hexo generate
$ hexo g
// 部署建站
$ hexo deploy
$ hexo d
// 去除缓存文件
$ hexo clean
```
这几个命令就能应付常用博客发布了。

# 创建Github.io

在我的Github中创建 [github名].github.io这个项目，比如像我的 [violetjack.github.io](https://github.com/violetjack/violetjack.github.io) 。

# 上传博客配置

如果是通过git开源发布的，那么只需要在hexo项目根目录的 `_config.yml` 文件中添加如下配置：
```
deploy:
  type: git
  repo: [github.io 仓库]
  branch: [发布的分支]
  message: [发布消息]
```

# 主题

hexo搭建的博客有很多的主题样式，可以在 [这里](https://hexo.io/themes/) 查看选择。安装过程里面都会说。

比如我们安装 `Ada` 主题，首先用git克隆下仓库。这里，可以在hexo博客项目中去执行克隆行为，直接下载到hexo项目的themes目录下。
```
git clone https://github.com/shuiRong/hexo-theme-Ada.git themes/Ada
```
有些主题需要安装依赖库，在hexo项目根目录中安装：
```
npm install hexo-renderer-jade --save
```
最后，修改hexo项目根目录下 `_config.yml` 中的 theme 选项
```
theme: Ada
```
这就完成了主题的修改。
主题的配置工作呢，在 `./themes/Ada/_config.yml` 中，具体修改看相应的 Github README。
其实如果有任何对主题不满意的地方可以直接去主题中修改，代码并不难，如果只是想改几个文本全局搜一下就能搜到了。

# 添加关于页面
样式中一般只有首页和文章两个标签可用，如果我们想添加其他标签，如 关于我，该怎么办呢？
创建 关于我 页面（添加 layout 选项，默认为post）
```
$ hexo new page about
```
这样，项目中就多了 about 这个文件夹，修改其中的 md 文件即可编辑关于我页面。
然后将主题的配置 `./themes/Ada/_config.yml` 中的页面链接指向 about 即可。
```
# Header
menu:
  首页: /
  文章: /archives
  关于: /about
```

好啦，这里就简单介绍下Hexo的用法~主要是记录下搭建的过程。整理下步骤：
* 搭建环境
* 创建 Github.io，或者说GithubPage
* 使用hexo搭建博客
* 选择样式，添加页面、添加文章内容。
* 发布

就这么多啦~快去选择一个喜欢的样式做一个自己的博客，提升逼格把~
**最后展示一下我的博客：**[Vue实验室](https://violetjack.github.io/)

# 参考资料
[hexo中文网](https://hexo.io/zh-cn/docs/)
[github page](https://pages.github.com/)