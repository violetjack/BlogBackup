---
title: element 源码学习一 —— 认识框架
date: 2018-03-10
tag: "element源码学习"
---

> 由于面试需要，先来几发 element 源码学习博客。Vue 源码还将继续更新。

好，现在我们开始学习 element —— 最受欢迎的 Vue UI 框架。

# package.json

我觉得要看一个前端项目，首先必须得看看 `package.json` 这个文件。

## 编译入口

来看看编译的入口

```json
  "scripts": {
    # 安装依赖
    "bootstrap": "yarn || npm i",
    # 构建文件
    "build:file": "node build/bin/iconInit.js & node build/bin/build-entry.js & node build/bin/i18n.js & node build/bin/version.js",
    # 构建样式
    "build:theme": "node build/bin/gen-cssfile && gulp build --gulpfile packages/theme-chalk/gulpfile.js && cp-cli packages/theme-chalk/lib lib/theme-chalk",
    # 构建工具
    "build:utils": "cross-env BABEL_ENV=utils babel src --out-dir lib --ignore src/index.js",
    # 构建umd
    "build:umd": "node build/bin/build-locale.js"
  }
```
这里我们以看源码的角度，先了解构建文件命令。其实就是 node 执行了几个 js 脚本。我们深入看下 `iconInit`、 `build-entry`、 `i18n`、 `version` 这些脚本文件。

```js
// build/bin/build-all.js
'use strict';

const components = require('../../components.json');
const execSync = require('child_process').execSync;
const existsSync = require('fs').existsSync;
const path = require('path');

let componentPaths = [];

delete components.index;
delete components.font;

// 遍历 components 的 key，找到相应 component 的路径，将路径保存到 componentPaths 数组
Object.keys(components).forEach(key => {
  const filePath = path.join(__dirname, `../../packages/${key}/cooking.conf.js`);

  if (existsSync(filePath)) {
    componentPaths.push(`packages/${key}/cooking.conf.js`);
  }
});

// pathA,pathB,pathC
const paths = componentPaths.join(',');
// 拼接为 shell 命令，并调用 execSync 方法执行。
const cli = path.join('node_modules', '.bin', 'cooking') + ` build -c ${paths} -p`;

execSync(cli, {
  stdio: 'inherit'
});

```
以上方法主要是获取所有组件名，然后拼接为 shell 命令，执行 shell 命令进行 build。

```js
// build/bin/iconInit.js
'use strict';

var postcss = require('postcss');
var fs = require('fs');
var path = require('path');
var fontFile = fs.readFileSync(path.resolve(__dirname, '../../packages/theme-chalk/src/icon.scss'), 'utf8');
var nodes = postcss.parse(fontFile).nodes;
var classList = [];
// 遍历匹配正则，符合则传入到数组中
nodes.forEach((node) => {
  var selector = node.selector || '';
  var reg = new RegExp(/\.el-icon-([^:]+):before/); // 正则： .el-icon-(多个非:字符):before
  var arr = selector.match(reg);

  if (arr && arr[1]) {
    classList.push(arr[1]);
  }
});
// 导出 icon.json 文件
fs.writeFile(path.resolve(__dirname, '../../examples/icon.json'), JSON.stringify(classList));
```
以上方法通过解析 icon.scss 最终导出 icon.json 文件，该文件保存了各种图标。

```js
// build/bin/i18n.js
'use strict';

var fs = require('fs');
var path = require('path');
// 获取 page.json
var langConfig = require('../../examples/i18n/page.json');

langConfig.forEach(lang => {
  try {
    // 获取文件信息
    fs.statSync(path.resolve(__dirname, `../../examples/pages/${ lang.lang }`));
  } catch (e) {
    // 创建文件夹
    fs.mkdirSync(path.resolve(__dirname, `../../examples/pages/${ lang.lang }`));
  }

  // 遍历写入文件
  Object.keys(lang.pages).forEach(page => {
    var templatePath = path.resolve(__dirname, `../../examples/pages/template/${ page }.tpl`);
    var outputPath = path.resolve(__dirname, `../../examples/pages/${ lang.lang }/${ page }.vue`);
    var content = fs.readFileSync(templatePath, 'utf8');
    var pairs = lang.pages[page];

    Object.keys(pairs).forEach(key => {
      content = content.replace(new RegExp(`<%=\\s*${ key }\\s*>`, 'g'), pairs[key]);
    });

    fs.writeFileSync(outputPath, content);
  });
});
```
以上代码是国际化的过程，最终将会在 `examples/pages/` 目录中生成不同语言的内容。国际化具体内容请参照 [国际化](http://element-cn.eleme.io/#/zh-CN/component/i18n)。

![生成结果](https://upload-images.jianshu.io/upload_images/1987062-5b0354332c689f31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```js
// build/bin/version.js
var fs = require('fs');
var path = require('path');
var version = process.env.VERSION || require('../../package.json').version;
var content = { '1.4.13': '1.4', '2.0.11': '2.0', '2.1.0': '2.1' };
if (!content[version]) content[version] = '2.2';
fs.writeFileSync(path.resolve(__dirname, '../../examples/versions.json'), JSON.stringify(content));
```
获取 version，定义了一个 content，如果当前版本不在 content 中，那么再添加一个版本数据。由于我学习的版本是 `2.2.1`，最终生成的结果是:
```json
{"1.4.13":"1.4","2.0.11":"2.0","2.1.0":"2.1","2.2.1":"2.2"}
```
出现这几个版本号的原因么，看下官网就能发现端倪，应该是几个重要的稳定版本。

![版本号](https://upload-images.jianshu.io/upload_images/1987062-e863a98c9731821b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

来看下 `build-entry.js` 文件
```js
// build/bin/build-entry.js
var Components = require('../../components.json'); // 组件数据
var fs = require('fs'); // node文件系统
var render = require('json-templater/string');
var uppercamelcase = require('uppercamelcase'); // 驼峰大小写写法
var path = require('path'); // node路径系统
var endOfLine = require('os').EOL;

// 导出路径
var OUTPUT_PATH = path.join(__dirname, '../../src/index.js');

// 导入template、安装组件template、主要template
var IMPORT_TEMPLATE = 'import {{name}} from \'../packages/{{package}}/index.js\';';
var INSTALL_COMPONENT_TEMPLATE = '  {{name}}';
var MAIN_TEMPLATE = `/* Automatically generated by './build/bin/build-entry.js' */

{{include}}
import locale from 'element-ui/src/locale';
import CollapseTransition from 'element-ui/src/transitions/collapse-transition';

const components = [
{{install}},
  CollapseTransition
];

const install = function(Vue, opts = {}) {
  locale.use(opts.locale);
  locale.i18n(opts.i18n);

  components.map(component => {
    Vue.component(component.name, component);
  });

  Vue.use(Loading.directive);

  const ELEMENT = {};
  ELEMENT.size = opts.size || '';

  Vue.prototype.$loading = Loading.service;
  Vue.prototype.$msgbox = MessageBox;
  Vue.prototype.$alert = MessageBox.alert;
  Vue.prototype.$confirm = MessageBox.confirm;
  Vue.prototype.$prompt = MessageBox.prompt;
  Vue.prototype.$notify = Notification;
  Vue.prototype.$message = Message;

  Vue.prototype.$ELEMENT = ELEMENT;
};

/* istanbul ignore if */
if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue);
}

module.exports = {
  version: '{{version}}',
  locale: locale.use,
  i18n: locale.i18n,
  install,
  CollapseTransition,
  Loading,
{{list}}
};

module.exports.default = module.exports;
`;

delete Components.font;
// 组件名
var ComponentNames = Object.keys(Components);

var includeComponentTemplate = [];
var installTemplate = [];
var listTemplate = [];

// 遍历组件名解析template
ComponentNames.forEach(name => {
  var componentName = uppercamelcase(name); // 驼峰命名

  includeComponentTemplate.push(render(IMPORT_TEMPLATE, {
    name: componentName,
    package: name
  }));

  if (['Loading', 'MessageBox', 'Notification', 'Message'].indexOf(componentName) === -1) {
    installTemplate.push(render(INSTALL_COMPONENT_TEMPLATE, {
      name: componentName,
      component: name
    }));
  }

  if (componentName !== 'Loading') listTemplate.push(`  ${componentName}`);
});
// 主要template
var template = render(MAIN_TEMPLATE, {
  include: includeComponentTemplate.join(endOfLine),
  install: installTemplate.join(',' + endOfLine),
  version: process.env.VERSION || require('../../package.json').version,
  list: listTemplate.join(',' + endOfLine)
});
// 导出文件
fs.writeFileSync(OUTPUT_PATH, template);
console.log('[build entry] DONE:', OUTPUT_PATH);
```
以上代码中，先是定义了三个 template，然后使用 render 方法来渲染这些 template。最后生成一个主要 template 导出为文件。render 函数中的第二个参数为 template 中 `{{name}}` 的数据。
这个 render 方法来自 [json-templater](https://www.npmjs.com/package/json-templater) 库，这个库可以将字符串编译为 js 代码。

## 依赖关系

看看 element 都 depend 了些什么？下面对 element 的依赖作了注释。

```json
  "dependencies": {
    // 异步验证器
    "async-validator": "~1.8.1",
    // vue和jsx合并参数的语法转译器？
    "babel-helper-vue-jsx-merge-props": "^2.0.0",
    // 深入合并
    "deepmerge": "^1.2.0",
    // 鼠标滚轮在多个浏览器之间的标准化。
    "normalize-wheel": "^1.0.1",
    // 方法的 Throttle/debounce？ https://www.npmjs.com/package/throttle-debounce
    "throttle-debounce": "^1.0.1"
  },
  "peerDependencies": {
    // vue核心源码
    "vue": "^2.5.2"
  },
  "devDependencies": {
    // 一个托管的全文、数字和分面搜索引擎，能够在第一次按键时提供实时结果。
    "algoliasearch": "^3.24.5",
    // babel
    "babel-cli": "^6.14.0",
    "babel-core": "^6.14.0",
    "babel-loader": "^6.2.5",
    "babel-plugin-add-module-exports": "^0.2.1",
    "babel-plugin-module-resolver": "^2.2.0",
    "babel-plugin-syntax-jsx": "^6.8.0",
    "babel-plugin-transform-vue-jsx": "^3.3.0",
    "babel-preset-es2015": "^6.14.0",
    // chai断言库
    "chai": "^3.5.0",
    // 为服务器专门设计的核心jQuery的快速、灵活、精益的实现。
    "cheerio": "^0.18.0",
    // node fs工具
    "chokidar": "^1.7.0",
    // cooking前端构建工具
    "cooking": "^1.5.4",
    "cooking-lint": "0.1.3",
    // 继承vue2配置项的cooking插件
    "cooking-vue2": "^0.3.3",
    // 复制webpack插件
    "copy-webpack-plugin": "^4.1.1",
    // 代码测试覆盖率
    "coveralls": "^2.11.14",
    // 跨平台支持UNIX命令
    "cp-cli": "^1.0.2",
    // 运行在平台上设置和使用环境变量的脚本。
    "cross-env": "^3.1.3",
    // css 加载器
    "css-loader": "^0.28.7",
    // es6 promise支持
    "es6-promise": "^4.0.5",
    // eslint语法检测
    "eslint": "4.14.0",
    "eslint-config-elemefe": "0.1.1",
    "eslint-loader": "^1.9.0",
    "eslint-plugin-html": "^4.0.1",
    "eslint-plugin-json": "^1.2.0",
    "extract-text-webpack-plugin": "^3.0.1",
    // 文件加载和保存
    "file-loader": "^1.1.5",
    "file-save": "^0.2.0",
    // 将文件发布到github的 gh-pages 分支
    "gh-pages": "^0.11.0",
    // gulp打包
    "gulp": "^3.9.1",
    "gulp-autoprefixer": "^4.0.0",
    "gulp-cssmin": "^0.1.7",
    "gulp-postcss": "^6.1.1",
    "gulp-sass": "^3.1.0",
    // js 高亮
    "highlight.js": "^9.3.0",
    // html加载器
    "html-loader": "^0.5.1",
    // html webpack插件
    "html-webpack-plugin": "^2.30.1",
    // A Webpack loader for injecting code into modules via their dependencies
    "inject-loader": "^3.0.1",
    // isparta instrumenter loader for webpack，用于测试
    "isparta-loader": "^2.0.0",
    // json加载器
    "json-loader": "^0.5.7",
    // json和js的模板生成工具
    "json-templater": "^1.0.4",
    // karma测试库
    "karma": "^1.3.0",
    "karma-chrome-launcher": "^2.2.0",
    "karma-coverage": "^1.1.1",
    "karma-mocha": "^1.2.0",
    "karma-sinon-chai": "^1.2.4",
    "karma-sourcemap-loader": "^0.3.7",
    "karma-spec-reporter": "0.0.26",
    "karma-webpack": "^1.8.0",
    // 用于管理具有多个包的JavaScript项目的工具。
    "lerna": "^2.0.0-beta.32",
    // 模拟时间工具
    "lolex": "^1.5.1",
    // markdown解析器
    "markdown-it": "^6.1.1",
    "markdown-it-anchor": "^2.5.0",
    "markdown-it-container": "^2.0.0",
    // mocha测试库
    "mocha": "^3.1.1",
    // node.js 的 sass
    "node-sass": "^4.5.3",
    // 视差滚动 https://perspective.js.org/#/zh-cn/
    "perspective.js": "^1.0.0",
    // postcss
    "postcss": "^5.1.2",
    "postcss-loader": "0.11.1",
    "postcss-salad": "^1.0.8",
    // node深度删除模块
    "rimraf": "^2.5.4",
    // sass加载器
    "sass-loader": "^6.0.6",
    // sinon测试框架
    "sinon": "^1.17.6",
    "sinon-chai": "^2.8.0",
    // 样式加载器
    "style-loader": "^0.19.0",
    // utf-8 字符转换
    "transliteration": "^1.1.11",
    // 驼峰写法
    "uppercamelcase": "^1.1.0",
    "url-loader": "^0.6.2",
    // vue
    "vue": "^2.5.2",
    "vue-loader": "^13.3.0",
    "vue-markdown-loader": "1",
    "vue-router": "2.7.0",
    "vue-template-compiler": "^2.5.2",
    "vue-template-es2015-compiler": "^1.6.0",
    // webpack
    "webpack": "^3.7.1",
    "webpack-dev-server": "^2.9.1",
    "webpack-node-externals": "^1.6.0"
  }
```

阿西吧，这依赖库真心多~~不知道他们如何找到这么多库的。

# src目录

再来看看项目结构部分。按常理源码肯定是放在 `src` 目录中的，我们找到 `src/index.js`。代码有点长，只贴出 `install` 方法部分了。说下都干了什么：导入所有组件，定义安装方法，判断环境执行 `install` 方法，最后整体导出。
```js
const install = function(Vue, opts = {}) {
  locale.use(opts.locale);
  locale.i18n(opts.i18n);

  components.map(component => {
    // 遍历将组件加入到Vue中
    Vue.component(component.name, component);
  });

  // 加载中
  Vue.use(Loading.directive);

  const ELEMENT = {};
  ELEMENT.size = opts.size || '';

  // 定义Vue的原型 prototype
  Vue.prototype.$loading = Loading.service;
  Vue.prototype.$msgbox = MessageBox;
  Vue.prototype.$alert = MessageBox.alert;
  Vue.prototype.$confirm = MessageBox.confirm;
  Vue.prototype.$prompt = MessageBox.prompt;
  Vue.prototype.$notify = Notification;
  Vue.prototype.$message = Message;

  Vue.prototype.$ELEMENT = ELEMENT;
};

/* istanbul ignore if */
if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue);
}
```
从组件的导入 `import Button from '../packages/button/index.js';` 可以看到所有组件都是在 packages 目录下的。这部分我们会在之后重点学习。
那么问题来了，既然组件都在 `packages` 中了，那么 `src` 目录下都干了些什么呢？来看看各个目录的功能：

* directive 实现滚轮优化和避免重复点击。
* locale 用于 i18n 国际化功能。
* mixins 看样子应该是用于混合到 Vue 实例的 options 中的。
* transition 在渲染是操作style做过渡效果处理。
* utils 工具文件夹。

主要目的是项目结构，就不深入展开了。如有需要后面再讲。

# 其他目录

上面说过，package 目录中存放了所有 component 组件的代码。另外也存放了组件的样式 `.scss` 文件。
而对于type目录中，存放的 `.ts` 文件。都是 TypeScript 文件。但是有个问题，我不太清楚这些 `.ts` 都用在何处。而且在 package.json 中也未导入 TypeScript 的库，只是在更新日志中有 `新增 TypeScript 类型声明` 这么一句话。这点有所疑惑。
test 目录下是各个组件的单元测试用例，这部分是学习单元测试写法的很好的参考代码（我学习测试框架就是在这里学的）。需要学习单元测试的可以深入看看。
example 目录下是 element 的示例项目。我们的目的是学习源码，所以这部分先忽略~

# 最后

简单了解了下项目的编译、项目的依赖库情况、项目的机构。下一篇开始学习一些组件的实现。逐步深入扒开element的神秘面纱。
