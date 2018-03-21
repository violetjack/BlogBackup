---
title: element 源码学习三 —— select 组件源码学习
date: 2018-03-20
tag: "element源码学习"
---

> select 选择器是个比较复杂的组件了，通过不同的配置可以有多种用法。有必要单独学习学习。

# 整体结构

以下是 select 的 template 结构，已去掉了一部分代码便于查看整体结构：
```html
<template>
  <div>
    <!-- 多选 -->
    <div
      v-if="multiple"
      ref="tags">
      <!-- collapse tags 多选时是否将选中值按文字的形式展示 -->
      <span v-if="collapseTags && selected.length">
        <el-tag
          type="info"
          disable-transitions>
          <span class="el-select__tags-text">{{ selected[0].currentLabel }}</span>
        </el-tag>
        <el-tag
          v-if="selected.length > 1"
          type="info"
          disable-transitions>
          <span class="el-select__tags-text">+ {{ selected.length - 1 }}</span>
        </el-tag>
      </span>
      <!-- 多选，多个 el-tag 组成 -->
      <transition-group @after-leave="resetInputHeight" v-if="!collapseTags">
        <el-tag
          v-for="item in selected"
          :key="getValueKey(item)"
          type="info"
          disable-transitions>
          <span class="el-select__tags-text">{{ item.currentLabel }}</span>
        </el-tag>
      </transition-group>
      <!-- 可输入文本的查询框 -->
      <input
        v-model="query"
        v-if="filterable"
        ref="input">
    </div>
    <!-- 显示结果框 read-only -->
    <el-input
      ref="reference"
      v-model="selectedLabel">
      <!-- 用户显示清空和向下箭头 -->
      <i slot="suffix"></i>
    </el-input>
    <!-- 下拉菜单 -->
    <transition>
      <el-select-menu
        ref="popper"
        v-show="visible && emptyText !== false">
        <el-scrollbar
          tag="ul"
          wrap-class="el-select-dropdown__wrap"
          view-class="el-select-dropdown__list"
          ref="scrollbar"
          v-show="options.length > 0 && !loading">
          <!-- 默认项（创建条目） -->
          <el-option
            :value="query"
            created
            v-if="showNewOption">
          </el-option>
          <!-- 插槽，用于放 option 和 option-group -->
          <slot></slot>
        </el-scrollbar>
        <!-- loading 加载中文本 -->
        <p
          v-if="emptyText &&
            (!allowCreate || loading || (allowCreate && options.length === 0 ))">
          {{ emptyText }}
        </p>
      </el-select-menu>
    </transition>
  </div>
</template>
```

具体都写在注释中了~从上面内容中可以看到，select 考虑了很多情况，如单选、多选、搜索、下拉框、图标等等。并且使用 slot 插槽来获取开发者传递的 option 和 option-group 组件。
可以发现在 select 中使用了多个外部组件，也就是说 el-select 是由多个组件组装成的一个复杂组件~

```js
  // components
  import ElInput from 'element-ui/packages/input';
  import ElSelectMenu from './select-dropdown.vue';
  import ElOption from './option.vue';
  import ElTag from 'element-ui/packages/tag';
  import ElScrollbar from 'element-ui/packages/scrollbar';
```

# select 要实现的功能

参照[官方文档](http://element-cn.eleme.io/#/zh-CN/component/select)的内容罗列出 select 的一些功能，后面跟上我对功能实现的理解：
* 单选 —— 点击 `select` 弹出下拉框，点击 `option` 完成赋值。
* 禁用 —— `select` 和 `option` 都有 `disabled` 选项用于禁用。
* 清空 —— 如果 `select` 中有内容，鼠标悬浮在 `input` 上显示删除图标，点击执行删除操作。
* 多选（平铺展示和数字显示数量两种方式） —— 参数 model 变为数组，点击下拉菜单中的选项添加或删除数组中的值。
* 自定义模板 —— option 中定义了 `slot` 插槽，默认加了 `span` 显示内容。可以修改 `el-option` 标签中内容来自定义模板。
* 分组 —— 使用 option-group 组件来实现分组效果。
* 搜索 —— 通过正则匹配搜索项，不符合搜索项的控制 v-show 隐藏
* 创建条目 —— 在 `select` 中添加额外 `option`（一般 `option` 都是通过 `slot` 插槽传递的），如允许创建条目，则显示这条 `option` ,`option` 的内容显示为查询内容。

# 从几个问题去看源码逻辑

## 如何实现基本单选功能？

分析下基本功能：点击 input，显示下拉菜单；鼠标选中一项 option，隐藏下拉菜单；input 中显示选中的结果。
所以这里看下显示内容的 input 都有些什么事件：
```
      @focus="handleFocus" // 处理 焦点
      @blur="handleBlur" // 处理 焦点 离开
      @keyup.native="debouncedOnInputChange"
      @keydown.native.down.stop.prevent="navigateOptions('next')" // 向下按键，移动到下一个 option
      @keydown.native.up.stop.prevent="navigateOptions('prev')" // 向上按键，移动到上一个 option
      @keydown.native.enter.prevent="selectOption" // 回车按键，选中option
      @keydown.native.esc.stop.prevent="visible = false"  // esc按键，隐藏下拉框
      @keydown.native.tab="visible = false" // tab按键，跳转到下一个文本框，隐藏下拉框
      @paste.native="debouncedOnInputChange" // 
      @mouseenter.native="inputHovering = true" // mouse enter 事件
      @mouseleave.native="inputHovering = false" // mouse leave 事件
```
从上面的这些事件中可以知道：选中方法为 `selectOption`（从英文字面意思都能知道~）；显示下拉框通过 `visible` 属性控制；以及其他按键的一些功能。这里主要主要看看 `selectOption` 方法。
```js
      selectOption() {
        if (!this.visible) {
          this.toggleMenu();
        } else {
          if (this.options[this.hoverIndex]) {
            this.handleOptionSelect(this.options[this.hoverIndex]);
          }
        }
      },
```
逻辑就是，如果下拉框未显示则执行 `toggleMenu` 方法触发下拉框，如果已显示下拉框则处理选择 option 的过程。看看这个 `toggleMenu` 方法：
```js
      toggleMenu() {
        if (!this.selectDisabled) {
          this.visible = !this.visible;
          if (this.visible) {
            (this.$refs.input || this.$refs.reference).focus();
          }
        }
      },
```
其实就是控制下拉菜单的显示和隐藏。如果显示的时候定焦在 `input` 和 `reference` 上，它们其实就是单选和多选的 input 框（多选 input 定义了 `ref="input"` 单选 input 定义了 `ref="reference"`）。
至此，下拉菜单的显示与隐藏解决了。然后我们去找 option 点击事件：
```js
      // 处理选项选中事件
      handleOptionSelect(option) {
        if (this.multiple) {
          // 多选
          const value = this.value.slice();
          const optionIndex = this.getValueIndex(value, option.value);
          if (optionIndex > -1) {
            // 已选中，从数组中移除
            value.splice(optionIndex, 1);
          } else if (this.multipleLimit <= 0 || value.length < this.multipleLimit) {
            // 未选中，传入数组
            value.push(option.value);
          }
          this.$emit('input', value);
          this.emitChange(value);
          if (option.created) {
            this.query = '';
            this.handleQueryChange('');
            this.inputLength = 20;
          }
          // 查询
          if (this.filterable) this.$refs.input.focus();
        } else {
          // 单选
          this.$emit('input', option.value);
          this.emitChange(option.value);
          this.visible = false;
        }
        // 渲染完成后
        this.$nextTick(() => {
          this.scrollToOption(option);
          this.setSoftFocus();
        });
      },
```
处理选中事件考虑了单选和多选两种情况。
如果是多选，检索选中 option 是否在 `value` 数组中，有则移除、无则添加到 `value` 数组中。然后 `$emit` 触发 `input` 事件，执行 `emitChange` 方法。如果 option 的 `created` 为 true，则清空查询内容。
如果是单选，`$emit` 触发 `input` 事件将选中值传递给父组件，执行 `emitChange` 方法，最后隐藏下拉菜单。
最后使用 `$nextTick` 方法处理下界面。
到这里，选中 option 后下拉菜单消失问题解决，只剩下显示结果到 input 中了。这个显示结果的过程是通过对 `visible` 属性的监听来完成的（一开始以为在 `emitChange` 结果发现那只是触发改变事件的）。
```js
      visible(val) {
        // 在下拉菜单隐藏时
        if (!val) {
          // 处理图标
          this.handleIconHide();
          // 广播下拉菜单销毁事件
          this.broadcast('ElSelectDropdown', 'destroyPopper');
          // 取消焦点
          if (this.$refs.input) {
            this.$refs.input.blur();
          }
          // 重置过程
          this.query = '';
          this.previousQuery = null;
          this.selectedLabel = '';
          this.inputLength = 20;
          this.resetHoverIndex();
          this.$nextTick(() => {
            if (this.$refs.input &&
              this.$refs.input.value === '' &&
              this.selected.length === 0) {
              this.currentPlaceholder = this.cachedPlaceHolder;
            }
          });
          // 如果不是多选，进行赋值现在 input 中
          if (!this.multiple) {
            // selected 为当前选中的 option
            if (this.selected) {
              if (this.filterable && this.allowCreate &&
                this.createdSelected && this.createdOption) {
                this.selectedLabel = this.createdLabel;
              } else {
                this.selectedLabel = this.selected.currentLabel;
              }
              // 查询结果
              if (this.filterable) this.query = this.selectedLabel;
            }
          }
        } else {
          // 下拉菜单显示
          // 处理图片显示
          this.handleIconShow();
          // 广播下拉菜单更新事件
          this.broadcast('ElSelectDropdown', 'updatePopper');
          // 处理查询事件
          if (this.filterable) {
            this.query = this.remote ? '' : this.selectedLabel;
            this.handleQueryChange(this.query);
            if (this.multiple) {
              this.$refs.input.focus();
            } else {
              if (!this.remote) {
                this.broadcast('ElOption', 'queryChange', '');
                this.broadcast('ElOptionGroup', 'queryChange');
              }
              this.broadcast('ElInput', 'inputSelect');
            }
          }
        }
        // 触发 visible-change 事件
        this.$emit('visible-change', val);
      },
```
从 template 中可知，显示结果的 input 绑定的 `v-model` 是 `selectedLabel`，而 select 是通过获取下拉菜单的显示与隐藏事件来执行结果显示部分的功能的。最终 `selectedLabel` 获得到了选中的 option 的 `label` 内容。
这样，从 **点击-单选-显示** 的流程就实现了。还是很简单的。

## 如何实现多选，多选选中后 option 右侧的勾以及 input 中的 tag 如何显示？

关于多选，在刚才讲单选的时候提及了一些了。所以有些代码就不贴出浪费篇幅了。具体逻辑如下：
先点击 input 执行 `selectOption` 方法显示下拉菜单，然后点击下拉菜单中的 option，执行 `handleOptionSelect` 方法将 option 的值都传给 `value` 数组。此时 `value` 数组改变，触发 watch 中的 `value` 变化监听方法。
```js
      value(val) {
        // 多选
        if (this.multiple) {
          this.resetInputHeight();
          if (val.length > 0 || (this.$refs.input && this.query !== '')) {
            this.currentPlaceholder = '';
          } else {
            this.currentPlaceholder = this.cachedPlaceHolder;
          }
          if (this.filterable && !this.reserveKeyword) {
            this.query = '';
            this.handleQueryChange(this.query);
          }
        }
        this.setSelected();
        // 非多选查询
        if (this.filterable && !this.multiple) {
          this.inputLength = 20;
        }
      },
```
以上代码关键是执行了 `setSelected` 方法：
```js
      // 设置选择项
      setSelected() {
        // 单选
        if (!this.multiple) {
          let option = this.getOption(this.value);
          // created 是指创建出来的 option，这里指 allow-create 创建的 option 项
          if (option.created) {
            this.createdLabel = option.currentLabel;
            this.createdSelected = true;
          } else {
            this.createdSelected = false;
          }
          this.selectedLabel = option.currentLabel;
          this.selected = option;
          if (this.filterable) this.query = this.selectedLabel;
          return;
        }
        // 遍历获取 option
        let result = [];
        if (Array.isArray(this.value)) {
          this.value.forEach(value => {
            result.push(this.getOption(value));
          });
        }
        // 赋值
        this.selected = result;
        this.$nextTick(() => {
          // 重置 input 高度
          this.resetInputHeight();
        });
      },
```
可以看到如果是多选，那么将 `value` 数组遍历，获取相应的 `option` 值，传给 `selected`。而多选界面其实就是对于这个 `selected` 的 v-for 遍历显示。显示的标签使用的是 element 的另外一个组件 [el-tag](http://element-cn.eleme.io/#/zh-CN/component/tag)
```html
        <el-tag
          v-for="item in selected"
          :key="getValueKey(item)">
          <span class="el-select__tags-text">{{ item.currentLabel }}</span>
        </el-tag>
```
这里顺便提一句： option 的 `created` 参数用于标识是 `select` 组件中创建的那个用于创建条目的 `option`。而从 slot 插槽传入的 option 是不用传 `created` 参数的。

## 如何实现搜索功能？

从 template 中可知，select 有两个 input，一个用于显示结果，一个则用于查询搜索。我们来看下搜索内容的 input 文本框如何实现搜索功能：
在 input 中有 `@input="e => handleQueryChange(e.target.value)"`这么一段代码。所以，handleQueryChange 方法就是关键所在了。
```js
      // 处理查询改变
      handleQueryChange(val) {
        if (this.previousQuery === val) return;
        if (
          this.previousQuery === null &&
          (typeof this.filterMethod === 'function' || typeof this.remoteMethod === 'function')
        ) {
          this.previousQuery = val;
          return;
        }
        this.previousQuery = val;
        this.$nextTick(() => {
          if (this.visible) this.broadcast('ElSelectDropdown', 'updatePopper');
        });
        this.hoverIndex = -1;
        if (this.multiple && this.filterable) {
          const length = this.$refs.input.value.length * 15 + 20;
          this.inputLength = this.collapseTags ? Math.min(50, length) : length;
          this.managePlaceholder();
          this.resetInputHeight();
        }
        if (this.remote && typeof this.remoteMethod === 'function') {
          this.hoverIndex = -1;
          this.remoteMethod(val);
        } else if (typeof this.filterMethod === 'function') {
          this.filterMethod(val);
          this.broadcast('ElOptionGroup', 'queryChange');
        } else {
          this.filteredOptionsCount = this.optionsCount;
          this.broadcast('ElOption', 'queryChange', val);
          this.broadcast('ElOptionGroup', 'queryChange');
        }
        if (this.defaultFirstOption && (this.filterable || this.remote) && this.filteredOptionsCount) {
          this.checkDefaultFirstOption();
        }
      },
```
其中，`remoteMethod` 和 `filterMethod` 方法是自定义的远程查询和本地过滤方法。如果没有自定义的这两个方法，则会触发广播给 `option` 和 `option-group` 组件 `queryChange` 方法。
```js
      // option.vue
      queryChange(query) {
        let parsedQuery = String(query).replace(/(\^|\(|\)|\[|\]|\$|\*|\+|\.|\?|\\|\{|\}|\|)/g, '\\$1');
        // 匹配字符决定是否显示当前option
        this.visible = new RegExp(parsedQuery, 'i').test(this.currentLabel) || this.created;
        if (!this.visible) {
          this.select.filteredOptionsCount--;
        }
      }
```
option 中通过正则匹配决定是否隐藏当前 option 组件，而 option-group 通过获取子组件，判断如果有子组件是可见的则显示，否则隐藏。
```js
      // option-group.vue
      queryChange() {
        this.visible = this.$children &&
          Array.isArray(this.$children) &&
          this.$children.some(option => option.visible === true);
      }
```
所以，其实 option 和 option-group 在搜索的时候只是隐藏掉了不匹配的内容而已。

## 下拉菜单的显示和隐藏效果是如何实现的？下拉菜单本质是什么东西？

下拉菜单是通过 [transition](https://cn.vuejs.org/v2/api/#transition) 来实现过渡动画的。
下拉菜单 `el-select-menu` 本质上就是一个 div 容器而已。
```html
  <div
    class="el-select-dropdown el-popper"
    :class="[{ 'is-multiple': $parent.multiple }, popperClass]"
    :style="{ minWidth: minWidth }">
    <slot></slot>
  </div>
```
另外，在代码中经常出现的通知下拉菜单显示和隐藏的广播在 `el-select-menu` 的 `mounted` 方法中接收使用：
```js
    mounted() {
      this.referenceElm = this.$parent.$refs.reference.$el;
      this.$parent.popperElm = this.popperElm = this.$el;
      this.$on('updatePopper', () => {
        if (this.$parent.visible) this.updatePopper();
      });
      this.$on('destroyPopper', this.destroyPopper);
    }
```

## 创建条目如何实现？

上文中提到过，就是在 select 中默认藏了一条 option，当创建条目时显示这个 option 并显示创建内容。点击这个 option 就可以把创建的内容添加到显示结果的 input 上了。

## 如何展示远程数据？

通过为 select 设置 `remote` 和 `remote-method` 属性来获取远程数据。`remote-method` 方法最终将数据赋值给 option 的 v-model 绑定数组数据将结果显示出来即可。

## 清空按钮显示和点击事件呢？
在显示结果的 input 文本框中有一个 `<i>` 标签，用于显示图标。
```html
      <!-- 用户显示清空和向下箭头 -->
      <i slot="suffix"
       :class="['el-select__caret', 'el-input__icon', 'el-icon-' + iconClass]"
       @click="handleIconClick"
      ></i>
```
最终 input 右侧显示什么图标由 `iconClass` 决定，其中 `circle-close` 就是圆形查查，即清空按钮~
```js
      iconClass() {
        let criteria = this.clearable &&
          !this.selectDisabled &&
          this.inputHovering &&
          !this.multiple &&
          this.value !== undefined &&
          this.value !== '';
        return criteria ? 'circle-close is-show-close' : (this.remote && this.filterable ? '' : 'arrow-up');
      },
```
`handleIconClick` 方法：
```js
      // 处理图标点击事件（删除按钮）
      handleIconClick(event) {
        if (this.iconClass.indexOf('circle-close') > -1) {
          this.deleteSelected(event);
        }
      },
      // 删除选中
      deleteSelected(event) {
        event.stopPropagation();
        this.$emit('input', '');
        this.emitChange('');
        this.visible = false;
        this.$emit('clear');
      },
```
最终，清空只是将文本清空掉并且关闭下拉菜单。其实当再次打开 select 的时候，option 还是选中在之前选中的那个位置，即 `HoverIndex` 没有变为 -1，不知道算不算 bug。

## option 的自定义模板是如何实现的？

很简单，使用了 slot 插槽。并且在 slot 中定义了默认显示方式。
```html
    <slot>
      <span>{{ currentLabel }}</span>
    </slot>
```

# 最后

第一次尝试用问题取代主题来写博客，这样看着中心是不是更明确一些？
最后，说下看完 select 组件的感受：
* element 通过自定义的广播方法进行父子组件间的通信。（好像以前Vue也有这个功能，后来弃用了。）
* 再复杂的组件都是由一个个基础的组件拼起来的。
* select 功能还是挺复杂的，加上子组件 1000+ 行代码了。本文只是讲了基本功能的实现，值得深入学习。
* 学习了高手写组件的方式和写法~之后在自己写组件的时候可以参考。
* 方法、参数命名非常规范，一眼就能看懂具体用法。
* 知道了 [Array.some()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some) 方法~

好吧，说好了一天写出来，结果断断续续花了三天才完成。有点高估自己能力啦~
说下之后的Vue实验室博客计划：计划再找两个复杂的 element 组件来学习，最后写一篇总结博客。然后试着自己去创建几个 UI 组件，学以致用。