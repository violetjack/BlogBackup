---
title: element源码学习二 —— 简单组件学习
date: 2018-03-11
tag: "element源码学习"
---

> 上一篇博客中学习了项目的结构，这篇博客来学几个简单的组件的实现。

在上一篇博客中我们提到了组件的源码都是存放在 `packages` 目录下的，所以我们从中挑一些组件来学习。先从简单的入手，来学习 button、radio、checkbox和InputNumber这四个组件的源码~

# Button
找到 `packages/button/` 目录下，先看看 index.js 做了什么？
```js
import ElButton from './src/button';

// 注册全局组件
ElButton.install = function(Vue) {
  Vue.component(ElButton.name, ElButton);
};

export default ElButton;
```
其实就是获取组件，然后全局注册组件。很好理解，我们自定义组件也经常这么干。看看组件内容吧~
```html
<!-- packages/button/src/button.vue -->
<template>
  <button
    class="el-button"
    @click="handleClick"
    :disabled="disabled || loading"
    :autofocus="autofocus"
    :type="nativeType"
    :class="[
      type ? 'el-button--' + type : '',
      buttonSize ? 'el-button--' + buttonSize : '',
      {
        'is-disabled': disabled,
        'is-loading': loading,
        'is-plain': plain,
        'is-round': round
      }
    ]"
  >
    <i class="el-icon-loading" v-if="loading"></i>
    <i :class="icon" v-if="icon && !loading"></i>
    <span v-if="$slots.default"><slot></slot></span>
  </button>
</template>
<script>
  export default {
    name: 'ElButton',
    // 获取父级组件 provide 传递下来的数据。
    inject: {
      elFormItem: {
        default: ''
      }
    },

    // 属性 http://element-cn.eleme.io/#/zh-CN/component/button
    props: {
      // 类型 primary / success / warning / danger / info / text
      type: {
        type: String,
        default: 'default'
      },
      // 尺寸 medium / small / mini
      size: String,
      // 图标类名
      icon: {
        type: String,
        default: ''
      },
      // 原生type属性 button / submit / reset
      nativeType: {
        type: String,
        default: 'button'
      },
      // 是否加载中状态
      loading: Boolean,
      // 是否禁用状态
      disabled: Boolean,
      // 是否朴素按钮
      plain: Boolean,
      // 是否默认聚焦
      autofocus: Boolean,
      // 是否圆形按钮
      round: Boolean
    },

    computed: {
      // elFormItem 尺寸获取
      _elFormItemSize() {
        return (this.elFormItem || {}).elFormItemSize;
      },
      // 按钮尺寸计算
      buttonSize() {
        return this.size || this._elFormItemSize || (this.$ELEMENT || {}).size;
      }
    },

    methods: {
      // 点击事件，使得组件的点击事件为 @click，与原生点击保持一致。
      handleClick(evt) {
        this.$emit('click', evt);
      }
    }
  };
</script>

```
代码已注释~其实 script 部分很简单：通过inject和props获取数据，计算方法计算尺寸，事件处理点击事件。关键点在于button中的几个class。说白了，其实这就是个原生的button组件，只是样式上有所不同。
样式文件都是存在 `packages/theme-chalk/` 目录下的。所以 button 的样式目录位于 `packages/theme-chalk/src/button.scss`。由于自己 CSS 非常渣，所以这步先跳过~

# Radio

来看下Radio。嗯……目录位置在 `packages/radio/` 目录下。index.js 文件和 button 是一样的 —— 导入组件、定义全局注册组件方法、导出。所以这边来看看 radio.vue 文件。
```html
<!-- packages/button/src/radio.vue -->
<template>
  <label
    class="el-radio"
    :class="[
      border && radioSize ? 'el-radio--' + radioSize : '',
      { 'is-disabled': isDisabled },
      { 'is-focus': focus },
      { 'is-bordered': border },
      { 'is-checked': model === label }
    ]"
    role="radio"
    :aria-checked="model === label"
    :aria-disabled="isDisabled"
    :tabindex="tabIndex"
    @keydown.space.stop.prevent="model = label"
  >
    <!-- radio 图标部分 -->
    <span class="el-radio__input"
      :class="{
        'is-disabled': isDisabled,
        'is-checked': model === label
      }"
    > 
      <!-- 图标效果 -->
      <span class="el-radio__inner"></span>
      <input
        class="el-radio__original"
        :value="label"
        type="radio"
        v-model="model"
        @focus="focus = true"
        @blur="focus = false"
        @change="handleChange"
        :name="name"
        :disabled="isDisabled"
        tabindex="-1"
      >
    </span>
    <!-- 文本内容 -->
    <span class="el-radio__label">
      <slot></slot>
      <template v-if="!$slots.default">{{label}}</template>
    </span>
  </label>
</template>
<script>
  import Emitter from 'element-ui/src/mixins/emitter';

  export default {
    name: 'ElRadio',
    // 混合选项
    mixins: [Emitter],

    inject: {
      elForm: {
        default: ''
      },

      elFormItem: {
        default: ''
      }
    },

    componentName: 'ElRadio',

    props: {
      // value值
      value: {},
      // Radio 的 value
      label: {},
      // 是否禁用
      disabled: Boolean,
      // 原生 name 属性
      name: String,
      // 是否显示边框
      border: Boolean,
      // Radio 的尺寸，仅在 border 为真时有效 medium / small / mini
      size: String
    },

    data() {
      return {
        focus: false
      };
    },
    computed: {
      // 向上遍历查询父级组件是否有 ElRadioGroup，即是否在按钮组中
      isGroup() {
        let parent = this.$parent;
        while (parent) {
          if (parent.$options.componentName !== 'ElRadioGroup') {
            parent = parent.$parent;
          } else {
            this._radioGroup = parent;
            return true;
          }
        }
        return false;
      },
      // 重新定义 v-model 绑定内容的 get 和 set
      model: {
        get() {
          return this.isGroup ? this._radioGroup.value : this.value;
        },
        set(val) {
          if (this.isGroup) {
            this.dispatch('ElRadioGroup', 'input', [val]);
          } else {
            this.$emit('input', val);
          }
        }
      },
      // elFormItem 尺寸
      _elFormItemSize() {
        return (this.elFormItem || {}).elFormItemSize;
      },
      // 计算 radio 尺寸，用于显示带有边框的radio的尺寸大小
      radioSize() {
        const temRadioSize = this.size || this._elFormItemSize || (this.$ELEMENT || {}).size;
        return this.isGroup
          ? this._radioGroup.radioGroupSize || temRadioSize
          : temRadioSize;
      },
      // 是否禁用，如果radioGroup禁用则按钮禁用。
      isDisabled() {
        return this.isGroup
          ? this._radioGroup.disabled || this.disabled || (this.elForm || {}).disabled
          : this.disabled || (this.elForm || {}).disabled;
      },
      // 标签索引 0 or -1
      tabIndex() {
        return !this.isDisabled ? (this.isGroup ? (this.model === this.label ? 0 : -1) : 0) : -1;
      }
    },

    methods: {
      // 处理 @change 事件，如果有按钮组，出发按钮组事件。
      handleChange() {
        this.$nextTick(() => {
          this.$emit('change', this.model);
          this.isGroup && this.dispatch('ElRadioGroup', 'handleChange', this.model);
        });
      }
    }
  };
</script>
```
从代码中可以看到，radio不止是个input那么简单。后面还会显示的文本内容。script 部分的逻辑是：通过 inject 和 props 获取传参，data记录 radio 的 focus 状态。compute 中计算一些属性值；methods中处理onchange事件。radio 中多了些关于 radioGroup 的处理。具体我都写在注释中了。
另外要注意的还是 template 中的各类 class，radio的css文件想必也猜到了，文件位于 `packages/theme-chalk/src/radio.scss`。**到这里就会发现：这些简单组件主要是样式上面的处理，对于逻辑上处理并不大。**
在 radio 中导入了 `element-ui/src/mixins/emitter.js` 文件，来看看它的作用是什么？
```js
// src/mixins/emitter.js
// 广播
function broadcast(componentName, eventName, params) {
  // 遍历子组件
  this.$children.forEach(child => {
    // 组件名
    var name = child.$options.componentName;

    if (name === componentName) {
      // 触发事件
      child.$emit.apply(child, [eventName].concat(params));
    } else {
      // 执行broadcast方法
      broadcast.apply(child, [componentName, eventName].concat([params]));
    }
  });
}
export default {
  methods: {
    dispatch(componentName, eventName, params) {
      // 父级组件及其组件名
      var parent = this.$parent || this.$root;
      var name = parent.$options.componentName;

      // 有父级组件 同时 没有name 或者 name 不等于组件名
      while (parent && (!name || name !== componentName)) {
        // parent 向上获取父级组件
        parent = parent.$parent;

        if (parent) {
          name = parent.$options.componentName;
        }
      }
      // 触发 eventName 事件
      if (parent) {
        parent.$emit.apply(parent, [eventName].concat(params));
      }
    },
    broadcast(componentName, eventName, params) {
      broadcast.call(this, componentName, eventName, params);
    }
  }
};
```
由注释可见，dispatch 方法向上获取父级组件并触发 eventName 事件。broadcast 方法向下遍历子组件触发 eventName 事件。

至于 Vue 的 mixins 属性是干嘛的？

> 混入 (mixins) 是一种分发 Vue 组件中可复用功能的非常灵活的方式。混入对象可以包含任意组件选项。当组件使用混入对象时，所有混入对象的选项将被混入该组件本身的选项。

好啦，至于 radio 的 scss 解析？抱歉，暂时对scss不熟，之后补上~本篇关键将逻辑吧。

# checkbox

找到 checkbox 的项目目录，index.js 逻辑是一样的~所以只需看看 checkbox.vue 文件
```html
<!-- packages/button/src/checkbox.vue -->
<template>
  <label
    class="el-checkbox"
    :class="[
      border && checkboxSize ? 'el-checkbox--' + checkboxSize : '',
      { 'is-disabled': isDisabled },
      { 'is-bordered': border },
      { 'is-checked': isChecked }
    ]"
    role="checkbox"
    :aria-checked="indeterminate ? 'mixed': isChecked"
    :aria-disabled="isDisabled"
    :id="id"
  >
    <!-- checkbox -->
    <span class="el-checkbox__input"
      :class="{
        'is-disabled': isDisabled,
        'is-checked': isChecked,
        'is-indeterminate': indeterminate,
        'is-focus': focus
      }"
       aria-checked="mixed"
    >
      <span class="el-checkbox__inner"></span>
      <!-- 如果有 true-value 或者 false-value -->
      <input
        v-if="trueLabel || falseLabel"
        class="el-checkbox__original"
        type="checkbox"
        :name="name"
        :disabled="isDisabled"
        :true-value="trueLabel"
        :false-value="falseLabel"
        v-model="model"
        @change="handleChange"
        @focus="focus = true"
        @blur="focus = false">
      <input
        v-else
        class="el-checkbox__original"
        type="checkbox"
        :disabled="isDisabled"
        :value="label"
        :name="name"
        v-model="model"
        @change="handleChange"
        @focus="focus = true"
        @blur="focus = false">
    </span>
    <!-- 文本内容 -->
    <span class="el-checkbox__label" v-if="$slots.default || label">
      <slot></slot>
      <template v-if="!$slots.default">{{label}}</template>
    </span>
  </label>
</template>
<script>
  import Emitter from 'element-ui/src/mixins/emitter';

  export default {
    name: 'ElCheckbox',
    // 混合 Emitter
    mixins: [Emitter],

    inject: {
      elForm: {
        default: ''
      },
      elFormItem: {
        default: ''
      }
    },

    componentName: 'ElCheckbox',

    data() {
      return {
        // checkbox model
        selfModel: false,
        // 焦点
        focus: false,
        // 超过限制？
        isLimitExceeded: false
      };
    },

    computed: {
      model: {
        // 获取model值
        get() {
          return this.isGroup
            ? this.store : this.value !== undefined
              ? this.value : this.selfModel;
        },
        // 设置 selfModel
        set(val) {
          // checkbox group 的set逻辑处理
          if (this.isGroup) {
            // 处理 isLimitExceeded
            this.isLimitExceeded = false;
            (this._checkboxGroup.min !== undefined &&
              val.length < this._checkboxGroup.min &&
              (this.isLimitExceeded = true));

            (this._checkboxGroup.max !== undefined &&
              val.length > this._checkboxGroup.max &&
              (this.isLimitExceeded = true));
            // 触发 ElCheckboxGroup 的 input 事件
            this.isLimitExceeded === false &&
            this.dispatch('ElCheckboxGroup', 'input', [val]);
          } else {
            // 触发当前组件 input 事件
            this.$emit('input', val);
            // 赋值
            this.selfModel = val;
          }
        }
      },
      // 是否选中
      isChecked() {
        if ({}.toString.call(this.model) === '[object Boolean]') {
          return this.model;
        } else if (Array.isArray(this.model)) {
          return this.model.indexOf(this.label) > -1;
        } else if (this.model !== null && this.model !== undefined) {
          return this.model === this.trueLabel;
        }
      },
      // 是否为按钮组
      isGroup() {
        let parent = this.$parent;
        while (parent) {
          if (parent.$options.componentName !== 'ElCheckboxGroup') {
            parent = parent.$parent;
          } else {
            this._checkboxGroup = parent;
            return true;
          }
        }
        return false;
      },
      // 判断 group，checkbox 的 value 获取
      store() {
        return this._checkboxGroup ? this._checkboxGroup.value : this.value;
      },
      // 是否禁用
      isDisabled() {
        return this.isGroup
          ? this._checkboxGroup.disabled || this.disabled || (this.elForm || {}).disabled
          : this.disabled || (this.elForm || {}).disabled;
      },
      // elFormItem 的尺寸
      _elFormItemSize() {
        return (this.elFormItem || {}).elFormItemSize;
      },
      // checkbox 尺寸，同样需要有边框才有效
      checkboxSize() {
        const temCheckboxSize = this.size || this._elFormItemSize || (this.$ELEMENT || {}).size;
        return this.isGroup
          ? this._checkboxGroup.checkboxGroupSize || temCheckboxSize
          : temCheckboxSize;
      }
    },

    props: {
      // value值
      value: {},
      // 选中状态的值（只有在checkbox-group或者绑定对象类型为array时有效）
      label: {},
      // 设置 indeterminate 状态，只负责样式控制
      indeterminate: Boolean,
      // 是否禁用
      disabled: Boolean,
      // 当前是否勾选
      checked: Boolean,
      // 原生 name 属性
      name: String,
      // 选中时的值
      trueLabel: [String, Number],
      // 没有选中时的值
      falseLabel: [String, Number],
      id: String, /* 当indeterminate为真时，为controls提供相关连的checkbox的id，表明元素间的控制关系*/
      controls: String, /* 当indeterminate为真时，为controls提供相关连的checkbox的id，表明元素间的控制关系*/
      // 是否显示边框
      border: Boolean,
      // Checkbox 的尺寸，仅在 border 为真时有效
      size: String
    },

    methods: {
      // 添加数据到model
      addToStore() {
        if (
          Array.isArray(this.model) &&
          this.model.indexOf(this.label) === -1
        ) {
          this.model.push(this.label);
        } else {
          this.model = this.trueLabel || true;
        }
      },
      // 处理 @change 事件，如果是 group 要处理 group 的 change 事件。
      handleChange(ev) {
        if (this.isLimitExceeded) return;
        let value;
        if (ev.target.checked) {
          value = this.trueLabel === undefined ? true : this.trueLabel;
        } else {
          value = this.falseLabel === undefined ? false : this.falseLabel;
        }
        this.$emit('change', value, ev);
        this.$nextTick(() => {
          if (this.isGroup) {
            this.dispatch('ElCheckboxGroup', 'change', [this._checkboxGroup.value]);
          }
        });
      }
    },
    created() {
      // 如果 checked 为 true，执行 addToStore 方法
      this.checked && this.addToStore();
    },
    mounted() { // 为indeterminate元素 添加aria-controls 属性
      if (this.indeterminate) {
        this.$el.setAttribute('aria-controls', this.controls);
      }
    }
  };
</script>
```
其实和radio逻辑差不多。从上面的内容中已知 emitter.js 用于触发子组件或父组件的事件，而参数上与radio也差不多。不同点有，在显示checkbox时，如果有true-babel 或 false-babel 属性和没有这两个属性显示的是不同的 checkbox(v-if,v-else)。其他都差不多。
具体代码注释即可。

# InputNumber
```html
<template>
  <div
    @dragstart.prevent
    :class="[
      'el-input-number',
      inputNumberSize ? 'el-input-number--' + inputNumberSize : '',
      { 'is-disabled': inputNumberDisabled },
      { 'is-without-controls': !controls },
      { 'is-controls-right': controlsAtRight }
    ]">
    <!-- 减法 -->
    <span
      class="el-input-number__decrease"
      role="button"
      v-if="controls"
      v-repeat-click="decrease"
      :class="{'is-disabled': minDisabled}"
      @keydown.enter="decrease">
      <i :class="`el-icon-${controlsAtRight ? 'arrow-down' : 'minus'}`"></i>
    </span>
    <!-- 加法 -->
    <span
      class="el-input-number__increase"
      role="button"
      v-if="controls"
      v-repeat-click="increase"
      :class="{'is-disabled': maxDisabled}"
      @keydown.enter="increase">
      <i :class="`el-icon-${controlsAtRight ? 'arrow-up' : 'plus'}`"></i>
    </span>
    <!-- el-input 内容 -->
    <el-input
      ref="input"
      :value="currentValue"
      :disabled="inputNumberDisabled"
      :size="inputNumberSize"
      :max="max"
      :min="min"
      :name="name"
      :label="label"
      @keydown.up.native.prevent="increase"
      @keydown.down.native.prevent="decrease"
      @blur="handleBlur"
      @focus="handleFocus"
      @change="handleInputChange">
      <!-- 占位符模板 -->
      <template slot="prepend" v-if="$slots.prepend">
        <slot name="prepend"></slot>
      </template>
      <template slot="append" v-if="$slots.append">
        <slot name="append"></slot>
      </template>
    </el-input>
  </div>
</template>
<script>
  import ElInput from 'element-ui/packages/input';
  import Focus from 'element-ui/src/mixins/focus';
  import RepeatClick from 'element-ui/src/directives/repeat-click';

  export default {
    name: 'ElInputNumber',
    // options 混合
    mixins: [Focus('input')],
    inject: {
      elForm: {
        default: ''
      },
      elFormItem: {
        default: ''
      }
    },
    // 自定义指令
    directives: {
      repeatClick: RepeatClick
    },
    components: {
      ElInput
    },
    props: {
      // 计数器步长
      step: {
        type: Number,
        default: 1
      },
      // 设置计数器允许的最大值	
      max: {
        type: Number,
        default: Infinity
      },
      // 设置计数器允许的最小值
      min: {
        type: Number,
        default: -Infinity
      },
      // 绑定值
      value: {},
      // 是否禁用计数器
      disabled: Boolean,
      // 计数器尺寸 large, small
      size: String,
      // 是否使用控制按钮
      controls: {
        type: Boolean,
        default: true
      },
      // 控制按钮位置 right
      controlsPosition: {
        type: String,
        default: ''
      },
      // 原生 name 属性
      name: String,
      // 输入框关联的label文字
      label: String
    },
    data() {
      return {
        // 当前值
        currentValue: 0
      };
    },
    watch: {
      value: {
        // 立即执行 get()
        immediate: true,
        handler(value) {
          let newVal = value === undefined ? value : Number(value);
          if (newVal !== undefined && isNaN(newVal)) return;
          if (newVal >= this.max) newVal = this.max;
          if (newVal <= this.min) newVal = this.min;
          this.currentValue = newVal;
          // 触发 @input 事件
          this.$emit('input', newVal);
        }
      }
    },
    computed: {
      // 最小禁用，无法再减
      minDisabled() {
        return this._decrease(this.value, this.step) < this.min;
      },
      // 最大禁用，无法再加
      maxDisabled() {
        return this._increase(this.value, this.step) > this.max;
      },
      // 精度
      precision() {
        const { value, step, getPrecision } = this;
        return Math.max(getPrecision(value), getPrecision(step));
      },
      // 按钮是否要显示于右侧
      controlsAtRight() {
        return this.controlsPosition === 'right';
      },
      // FormItem尺寸
      _elFormItemSize() {
        return (this.elFormItem || {}).elFormItemSize;
      },
      // 计算尺寸
      inputNumberSize() {
        return this.size || this._elFormItemSize || (this.$ELEMENT || {}).size;
      },
      // 获取禁用状态
      inputNumberDisabled() {
        return this.disabled || (this.elForm || {}).disabled;
      }
    },
    methods: {
      // 计算精度
      toPrecision(num, precision) {
        if (precision === undefined) precision = this.precision;
        return parseFloat(parseFloat(Number(num).toFixed(precision)));
      },
      // 获取精度
      getPrecision(value) {
        if (value === undefined) return 0;
        const valueString = value.toString();
        const dotPosition = valueString.indexOf('.');
        let precision = 0;
        if (dotPosition !== -1) {
          precision = valueString.length - dotPosition - 1;
        }
        return precision;
      },
      // 获取加法后的精度
      _increase(val, step) {
        if (typeof val !== 'number' && val !== undefined) return this.currentValue;

        const precisionFactor = Math.pow(10, this.precision);
        // Solve the accuracy problem of JS decimal calculation by converting the value to integer.
        return this.toPrecision((precisionFactor * val + precisionFactor * step) / precisionFactor);
      },
      // 获取减法后的精度
      _decrease(val, step) {
        if (typeof val !== 'number' && val !== undefined) return this.currentValue;

        const precisionFactor = Math.pow(10, this.precision);

        return this.toPrecision((precisionFactor * val - precisionFactor * step) / precisionFactor);
      },
      // 加法行为
      increase() {
        if (this.inputNumberDisabled || this.maxDisabled) return;
        const value = this.value || 0;
        const newVal = this._increase(value, this.step);
        this.setCurrentValue(newVal);
      },
      // 减法行为
      decrease() {
        if (this.inputNumberDisabled || this.minDisabled) return;
        const value = this.value || 0;
        const newVal = this._decrease(value, this.step);
        this.setCurrentValue(newVal);
      },
      // 处理 blur 和 focus
      handleBlur(event) {
        this.$emit('blur', event);
        this.$refs.input.setCurrentValue(this.currentValue);
      },
      handleFocus(event) {
        this.$emit('focus', event);
      },
      // 设置当前value
      setCurrentValue(newVal) {
        const oldVal = this.currentValue;
        if (newVal >= this.max) newVal = this.max;
        if (newVal <= this.min) newVal = this.min;
        if (oldVal === newVal) {
          // 执行 el-input 中的 setCurrentValue 方法
          this.$refs.input.setCurrentValue(this.currentValue);
          return;
        }
        // 触发事件，改变value
        this.$emit('change', newVal, oldVal);
        this.$emit('input', newVal);
        this.currentValue = newVal;
      },
      // 处理文本框变化
      handleInputChange(value) {
        const newVal = value === '' ? undefined : Number(value);
        if (!isNaN(newVal) || value === '') {
          this.setCurrentValue(newVal);
        }
      }
    },
    mounted() {
      // 更改 el-input 内部 input 的属性。
      let innerInput = this.$refs.input.$refs.input;
      innerInput.setAttribute('role', 'spinbutton');
      innerInput.setAttribute('aria-valuemax', this.max);
      innerInput.setAttribute('aria-valuemin', this.min);
      innerInput.setAttribute('aria-valuenow', this.currentValue);
      innerInput.setAttribute('aria-disabled', this.inputNumberDisabled);
    },
    updated() {
      // 更改 el-input 内部 input 的属性。
      let innerInput = this.$refs.input.$refs.input;
      innerInput.setAttribute('aria-valuenow', this.currentValue);
    }
  };
</script>
```
先看HTML，这个组件由两个span按钮当做加减按钮，一个 el-input 组件显示数字结果。但是，对于最后两个 template 的作用不是很明白，不知道何时使用。
再看JS，代码中导入了 el-input 组件、 Focus 方法和 RepeatClick 方法。这两个方法之后详述。看到代码中还是以props、inject、compute方法来处理传入的数据。和前三个组件不同的地方在于，一是引用了组件需要处理组件，在 mounted 和 updated 方法执行时修改 el-input 组件中 input 的属性；二是加入了加法和减法的方法逻辑：求精度、做加减法、处理最大最小值限制。
从CSS角度上来说，要处理加法、减法按钮的位置和样式等功能……SCSS方面的内容暂且一概不论。

顺便看看 Focus 和 RepeatClick 方法~
```js
// src/directives/repeat-click.js
import { once, on } from 'element-ui/src/utils/dom';
// on 添加监听事件
// once 监听一次事件

export default {
  bind(el, binding, vnode) {
    let interval = null;
    let startTime;
    // 执行表达式方法
    const handler = () => vnode.context[binding.expression].apply();
    // 清除interval
    const clear = () => {
      if (new Date() - startTime < 100) {
        handler();
      }
      clearInterval(interval);
      interval = null;
    };

    // 监听 mousedown 鼠标点击事件
    on(el, 'mousedown', (e) => {
      if (e.button !== 0) return;
      startTime = new Date();
      once(document, 'mouseup', clear);
      clearInterval(interval);
      // setInterval() 方法可按照指定的周期（以毫秒计）来调用函数或计算表达式。
      interval = setInterval(handler, 100);
    });
  }
};
```
其中导入的on和once方法类似于vue的方法，on用于监听事件，once监听一次事件。监听 mousedown 事件。如果触发每 100 毫秒执行一次 handler，如果监听到 mousedown 判断时间间隔如果短于100毫秒，执行 handler 方法。取消计时器并清空 interval。从而实现了重复点击的效果。

```js
// src/mixins/focus.js
export default function(ref) {
  return {
    methods: {
      focus() {
        this.$refs[ref].focus();
      }
    }
  };
};
```
很简单的方法，就是用于定位一个元素，执行它的 focus 方法。

所以说:**src 中除了 locale 目录用于国际化和 utils 工具目录外，其他目录下都是用于组件的一些配置上的.**如 mixins 目录用于 mixins 项,directives 目录下的用于 directives 项。

# 最后

额……由于对于 CSS 比较菜，所以暂且不误人子弟，待我学成后再补上这部分内容。
总结下几个组件给我的感觉：其实 UI 框架主要在 UI 二字上。所以其实讲 UI 框架重点应该放在 CSS 上。逻辑部分其实蛮简单的。毕竟是组件不是整个项目，组件就是要易用性、可扩展性。
**所以说组件的主要逻辑都是进行传参的处理和计算。**
element的组件模式上其实是定义好 `[component].vue` 组件文件，然后导入到用户项目 Vue 实例的 components 项中来使用，其实和自定义 component 基本原理差不多。
下一篇，来看看一些复杂的组件的代码逻辑实现~~