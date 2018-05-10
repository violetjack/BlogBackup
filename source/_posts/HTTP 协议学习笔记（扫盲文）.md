---
title: HTTP 协议学习笔记（扫盲文）
date: 2018-03-17
---

> 作为一个前端，了解下 HTTP 协议是很有必要的。

先说个题外话，从《跃迁》一书中提到，高手获取信息的方式 —— 只获取一手、二手资料。
我在学习 HTTP 协议的时候犯了的毛病是 —— 百度了一堆 HTTP 相关文章，内容不一、质量参差不齐，看得云里雾里。最后干脆去Google查 HTTP，发现在 MDN 上有详细的 HTTP 相关资料，于是抛弃三手、四手资料，直接去看 MDN 上的内容。
所以学习知识，必须得先认清资料的来源和专业程度，在知识源头学习可以提高学习的效率和准确性。

# 简介

> **HTTP是一种能够获取如 HTML 这样的网络资源的 **[protocol](https://developer.mozilla.org/en-US/docs/Glossary/protocol "protocol: A protocol is a system of rules that define how data is exchanged within or between computers.  Communications between devices require that the devices agree on the format of the data that is being exchanged. The set of rules that defines a format is called a protocol.")(通讯协议)。**它是在 Web 上进行数据交换的基础，是一种 client-server 协议，也就是说，请求通常是由像浏览器这样的接受方发起的。一个完整的Web文档通常是由不同的子文档拼接而成的，像是文本、布局描述、图片、视频、脚本等等。**

这里加上我个人的理解：HTTP 是一种在客户端和服务端之间传输信息的传输协议，或者说是一整套传输方案。所以，像 Headers、Cookie、Methods 等都是 HTTP 协议中的内容。用于更好的解决 client-server 传输信息中遇到的各种问题。

关于HTTP的更多历史信息，可以看到阮一峰老师的[HTTP 协议入门](http://www.ruanyifeng.com/blog/2016/08/http.html)（阮一峰老师写的文章还是那么清晰明了，向老师学习！）。

# TCP/IP 三次握手

HTTP 的传输协议是 TCP/IP 协议，该协议连接是需要进行三次握手的。
那么为什么必须是三次？
> 这个问题的本质是：信道不可靠，但是通信双发需要就某个问题达成一致。而要解决这个问题，无论你在消息中包含什么信息，三次通信是理论上的最小值。所以三次握手不是TCP本身的要求，而是为了满足"在不可靠信道上可靠地传输信息"这一需求所导致的。

以下列出三次握手具体步骤与理解：

* 客户端发送 SYN 报文给服务器端，进入 SYN_SEND 状态。 —— 客户端向服务器端发送消息，请求它的回应。
* 服务器收到 SYN 报文，回应一个 SYN ACK 报文，进入 SYN_RECV 状态。 —— 服务器端收到消息，采取回应行为，回复消息告诉客户端收到了。
* 客户端收到服务器 SYN 报文，回应一个 ACK 报文，进入连接状态。 —— 客户端收到消息，这时客户端收到响应，表明发送的数据有回信了。但是服务器端发送了回应却还不知道客户端有没有收到。这时客户端需要再次发送消息告诉服务器我收到了。服务器收到消息后，客户端和服务器都知道对方已准备好通讯，然后就开始连接通讯了。

# HTTP 的请求(request)和响应(response)
## 请求
请求为客户端发送给服务器的数据。具体有如下数据：
* 请求方法：如 GET、 POST 这类请求方法。
* 要获取的资源路径。
* HTTP协议版本号。
* Headers：传递附加信息。
* body：如果想 POST 请求，就会传递 body 资源数据给服务器。

![Request](https://upload-images.jianshu.io/upload_images/1987062-91b2abd5c2ae7063.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 响应
响应为服务器收到客户端发送数据返回的数据，具体有如下数据：
* HTTP协议版本号。
* 状态码（status code）：告知请求是否成功以及失败原因。
* 状态消息（status message）：非权威状态码信息，由服务器自行设定。
* Headers：传递附加信息。
* body： 响应返回的资源存在body中。一般返回图片、HTML等资源。

![Response](https://upload-images.jianshu.io/upload_images/1987062-f554dd2fc23359f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# Headers 头文件
头文件允许客户端和服务器通过请求和响应传递附加信息。
下面列出一些常用的消息头及其用法：

* Date 信息来源的日期时间
* Content-Type 指定服务器文档的MIME类型，帮助用户代理去处理接收到的数据。
* Content-Length 表示 body 的字节长度。
* Host 服务器的域名。
* User-Agent 可以用来识别发送请求的浏览器，是产品标记符和注释的清单。
* Accept 用户代理期望的MIME类型列表
* Accept-Encoding 列出用户代理支持的压缩方法
* Accept-Ranges 期望范围。参数：byte、none。
* Assess-Control-Allow-Origin 允许组织连接控制 。
* Age 对象在代理缓存中的时间
* Cache-Control 指定缓存机制
* Connection 是否保持网络连接打开状态。参数：keep-alive、close。
* ETag 特定版本资源标识符
* Expires 过期时间日期
* Server 服务器信息，如JSP、Apache等。
* Referer 可用于识别用户访问位置

还有一些自定专用消息头可通过 `X-` 前缀来添加。如 `x-oss-object-type`。

# 状态码
状态码是由服务器端发送的响应中带有的请求结果的信息码。可以分为以下几类：
* 1** 信息响应
* 2** 成功响应
* 3** 重定向
* 4** 客户端响应
* 5** 服务器响应

列一些常见的状态返回码：
* 200 请求成功。
* 304 未修改。
* 401 当前请求需要用户验证。
* 404 未找到资源。
* 500 服务器内部错误，无法完成请求。

状态码其实只要了解常用的，冷门的遇到的时候查查 MDN 就好了。

# Cookie
Cookie 是服务器发送到用户浏览器并保存在本地的一小块数据，它会在浏览器下次向同一服务器发起请求时被懈怠并发送到服务器上。
HTTP 本质上无状态，使用 Cookie 可以创建有状态会话。服务器指定 Cookie 后，浏览器的每次请求都会携带 Cookie 数据。所以，Cookie 常被用来验证用户登录信息。
更多 Cookie 内容请看 https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies

# 跨域

跨域就是获取不同域名下服务器的数据。跨域对于 GET 请求没有影响。对于其他请求，会先发送一个 OPTIONS 方法发送一个预检请求，获知服务端是否允许跨域请求。
所以，之前遇到 POST 方法请求变为 OPTIONS 请求报错的问题就是跨域问题。
在 Express 中也遇到个跨域问题，最终使用 `$ npm install cors` 的方式解决了。 

# HTTPS
## HTTP 和 HTTPS 的区别

因为 HTTP 所封装的信息是明文的，通过抓包工具可以分析其信息内容。所以 HTTP 是及其不安全的。
而 HTTPS 在 HTTP 传输协议的基础上加上了 SSL/TLS 加密协议，所以传输的信息都是加密过的。不易被截获信息内容。所以 HTTPS 比 HTTP 安全性更高。

## HTTPS 运行机制

大致运行步骤如下：
1. 客户端发起 HTTPS 请求
2. 服务端获取数字证书CA —— 服务器向数字证书认证机构申请获取数字证书 CA 表明服务器是合法的、无害的。
3. 传送数字证书 —— 将数字证书传给客户端
4. 客户端解析证书 —— 客户端向数字证书认证机构查询，验证服务器合法性。
5. 客户端传输加密后信息给服务端
6. 服务端解密信息
7. 服务端传输加密后的信息给客户端
8. 客户端解密信息

## 常见加密算法

非对称加密算法：RSA，DSA/DSS
对称加密算法：AES，RC4，3DES
HASH算法：MD5，SHA1，SHA256

# 最后

发现网上写 HTTP 协议的文章很多，但是众口不一，每篇博客的内容都有所不同，搜集资料的时候还是蛮困惑的。
本文整理了一些 HTTP 的基础知识，写的不够详细。更加深入的学习通过查 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP) 来解决问题啦~

# 参考资料
* MDN https://developer.mozilla.org/zh-CN/docs/Web/HTTP
* 通俗大白话来理解TCP协议的三次握手和四次分手https://github.com/jawil/blog/issues/14
* https原理：证书传递、验证和数据加密、解密过程解析
 http://blog.csdn.net/clh604/article/details/22179907

# 更新内容

本文只是 HTTP 的扫盲文，并没有深入学习 HTTP 的意思，其实深入学习 HTTP 水还是很深的。换位思考，我感觉面试官问 HTTP 相关问题的主要目的是：**能够很好与后端工程师沟通接口问题；更好处理前端工作中网络传输的问题；保证项目的信息安全；**前端工程师作为客户端方的开发，也有必要和服务器端的后端工程师合作将程序开发的高效、安全、易维护。
--03-19更新--
我去问了一位前辈前端为什么要学习 HTTP 协议，大牛前辈的回答是这样的：“**就前端来讲，两个用处吧：前端性能优化必须掌握，排障必须掌握**”

感觉这篇文章内容实在少了点。所以又补充了点内容，贴上几条最靠谱的 HTTP 学习资料希望能够帮到一些想深入学习 HTTP 童鞋~
* [《HTTP权威指南》](https://item.jd.com/11056556.html?dist=jd)（京东）
* [超文本传输协议 —— 维基百科](https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)
* [HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP)

记住文章开头所讲的，只去源头的知识学习才能保证学习的效率和准确性~祝我们能够更快更好的学习技术知识。