---
title: Radio, Checkbox 组件
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 48b7eeac
date: 2018-11-28 06:00:00
updated: 2018-11-28 06:00:00
---

antd 中的 Radio, Checkbox 组件均基于 rc-checkbox 实现。本文第一部分将介绍 [rc-checkbox](https://github.com/react-component/checkbox)，余下两部分再介绍 Radio, Checkbox 组件。

### rc-checkbox

rc-checkbox 输出 Checkbox 组件。其处理逻辑较为简单，包含样式类转换、多余 props 剔除等。元素层级上，rc-checkbox 使用 span 包裹 input 节点和内层 span 节点（该 span 节点将绘制可见的单选框样式）。样式类转换指 span 元素的样式类包含或可能包含 props.prefixCls, props.className, ${props.prefixCls}-checked, ${props.prefixCls}-disabled；input 节点的样式类为 ${props.prefixCls}-input；内层 span 节点的样式类为 ${props.prefixCls}-inner。多余 props 剔除指 input 元素只接受 name, id, type, readOnly, disabled, tabIndex, checked, onClick, onFocus, onBlur, onChange, autoFocus, value, ‘aria-‘, ‘data-‘, role 属性。其中，checked 由 Checkbox 组件的 state.checked 设定；onChange 设置为 Checkbox 组件的 handleChange 方法。当 input 节点数据变更时，props.onChange 接受到的数据先经由 Checkbox 组件转换。以下是其实现。

```javascript
handleChange = (e) => {
  const { props } = this;
  if (props.disabled) {
    return;
  }
  if (!('checked' in props)) {
    this.setState({
      checked: e.target.checked,
    });
  }
  props.onChange({
    target: {
      ...props,
      checked: e.target.checked,
    },
    stopPropagation() {
      e.stopPropagation();
    },
    preventDefault() {
      e.preventDefault();
    },
    nativeEvent: e.nativeEvent,
  });
};
```

### Radio 组件

antd 提供三种单选框组件：单选框 Radio, 单选框组合 RadioGroup, 单选框按钮 RadioButton。

Radio 组件的特殊处理逻辑为：通过 context.radioGroup = { name, onChange, value, disabled } 获得 RadioGroup 组件注入的数据，以与单选框组合完成交互。

Radio 组件的元素层级关系为 label 元素和 RcCheckbox 组件。以下 less 样式中，.@{radio-prefix-cls} 即 RcCheckbox 组件绘制的外层 span 元素；.@{radio-inner-prefix-cls} 即 RcCheckbox 组件绘制的内层 span 元素。Radio 组件呈现在视图上的样式以内层 span 元素及其 span:after 伪类内容绘制，选中时 span:after 设置 opacity: 1 样式。以下是 less 代码。

```less
.@{radio-prefix-cls} {
  &-inner {
    &:after {
      @radio-dot-size: @radio-size - 8px;
      position: absolute;
      width: @radio-dot-size;
      height: @radio-dot-size;
      left: (@radio-size - @radio-dot-size) / 2 - 1px;
      top: (@radio-size - @radio-dot-size) / 2 - 1px;
      border-radius: @radio-dot-size;
      display: table;
      border-top: 0;
      border-left: 0;
      content: ' ';
      background-color: @radio-dot-color;
      opacity: 0;
      transform: scale(0);
      transition: all @radio-duration @ease-in-out-circ;
    }

    position: relative;
    top: 0;
    left: 0;
    display: block;
    width: @radio-size;
    height: @radio-size;
    border-width: 1px;
    border-style: solid;
    border-radius: 100px;
    border-color: @border-color-base;
    background-color: @radio-button-bg;
    transition: all @radio-duration;
  }
}

.@{radio-prefix-cls}-checked {
  .@{radio-inner-prefix-cls} {
    border-color: @radio-dot-color;
    &:after {
      transform: scale(.875);
      opacity: 1;
      transition: all @radio-duration @ease-in-out-circ;
    }
  }
}
```

RadioButton 组件的渲染过程无甚特别之处，主要是默认将 props.prefixCls 设定为 ‘ant-radio-button’。

RadioGroup 组件接受 props.options 作为选项配置内容（props.options 数组项内容可以是字符串或对象）；并将 radioGroup 作为 context 内容传入子孙组件中。

### Checkbox

antd 提供两种复选框组件：复选框 Checkbox, 复选框组合 CheckboxGroup。

Checkbox 组件与 Radio 组件的实现无异，只是其接受的 context 数据内容为 checkboxGroup = { toggleOption, value, disabled } 属性。其中，toggleOption 用于影响 Checkbox 组件的 onChange 绑定函数，即改变 CheckboxGroup 组件的 state.value 状态；value 用于判断当前 Checkbox 组件是否处于选中状态。

CheckboxGroup 组件也与 RadioGroup 组件无异，只是其实现 toggleOption 方法，用于在复选框勾选或取消勾选时，实时更新 state.value 值，并调用 props.onChange 方法；并通过 context 注入到子孙组件中。