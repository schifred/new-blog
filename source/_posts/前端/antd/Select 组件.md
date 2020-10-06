---
title: Select 组件
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 6139fc9c
date: 2020-05-31 06:00:00
updated: 2020-05-31 06:00:00
---

Select 组件基于 [rc-select](https://github.com/react-component/select) 组装外观，主要功能包含：

* 多种展示模式：默认的单选模式、multiple 多选模式、tags 标签模式（支持随意输入值）、combobox 。
* 搜索功能：通过 props.showSearch、props.onSearch、props.optionFilterProp 配置搜索过滤能力。
* 选项分组：通过 OptGroup 组件设置选项分组。
* 扩展菜单：通过 props.dropdownRender 绘制扩展菜单。
* 值处理：props.labelInValue 能将 label 添加到提交数据中。

### 组件构成

![image](select.png)

* Select：由 [generate](https://github.com/react-component/select/blob/v10.4.0/src/generate.tsx#L207) 函数创建，整合 Selector、SelectTrigger、OptionList 处理逻辑。
* Selector：输入框，包含多选框、单选框两种模式。逻辑上包含点击聚焦事件、键盘事件、鼠标事件、手动录入事件、粘贴事件处理。其中，多选框会使用 rc-animate/lib/CSSMotionList 制作添加删除动效。
* SelectTrigger：基于 rc-trigger 处理弹层展示逻辑。
* OptionList：弹层内选择框，基于 rc-virtual-list 实现虚拟列表。
* Option、OptGroup：虚拟组件，其 props 作为选项、选项分组的属性。
* TransBtn：用于包装选择框中的后缀图标、多选框中的关清除图标。

以上组件大多基于 React.forwardRef 包装，以便设置 ref 引用；这些组件内部基于 React.useImperativeHandle 向上提供调用方法。

### 状态管理

#### value

1. 使用 React.useState 将 value 或 defaultValue 记录成内部状态值 innerValue。
2. innerValue 经 React.useRef 缓存成 prevValueRef，受控模式下使用 React.useEffect 更新 prevValueRef。
3. 如果存储的值包含 label，使用 React.useMemo 作纯 value 值提取，存储为 mergedRawValue。
4. 为了方便处理，使用 React.useMemo 将 mergedRawValue 转换为 Set 类型，存储为 rawValues。

内部 triggerChange 用于变更内部 innerValue，并与 props.onChange 完成对接（视条件将 label 传出外围）。

#### options

1. 使用 React.useMemo 缓存 props.options 或通过 children 获取选项，存储为 mergedOptions。tags 模式额外会将输入值转化成选项。
2. 使用 React.useMemo 缓存 Option、OptGroup 扁平化后的数据，存储为 mergedFlattenOptions。
3. 使用 React.useMemo 获取展示用的 displayOptions。如果在搜索模式下，会基于输入框的搜索值过滤选项列表。
4. 使用 React.useMemo 将 displayOptions 作扁平化处理，存储为 displayFlattenOptions。

内部 triggerSelect 用于传递选中项或取消选中项，传递的值中可能包含 label，与 props.onSelect、props.onDeselect 或 internalProps.onRawSelect、internalProps.onRawDeselect 完成对接。

#### searchValue

1. 使用 React.useState 制作内部状态值 innerSearchValue。
2. 实际作为搜索条件的 mergedSearchValue 可能是 innerSearchValue，或 props.value，或 props.searchValue，或 props.inputValue。

triggerSearch 最基本的实现是变更 innerSearchValue、对接 props.onSearch。当下拉框为 combobox 模式时，triggerSearch 同时会调用 triggerChange；当搜索由粘贴操作唤起时，它会调用 triggerChange、triggerSelect。

#### open

1. 使用 [useMergedState](https://github.com/react-component/util/blob/master/src/hooks/useMergedState.ts) 制作内部状态值 innerOpen。
2. 基于选项列表是否空值以及 innerOpen 获得 triggerOpen。

onToggleOpen 变更 popup 的显隐状态，并与 props.onDropdownVisibleChange 完成交互。

[useSelectTriggerControl](https://github.com/react-component/select/blob/v10.4.0/src/hooks/useSelectTriggerControl.ts) 用于实现：当 popup 展示时，点击 document 节点隐藏 popup 的逻辑。

#### onInternalSelect

基于 triggerChange、triggerSelect 等变更 Select 的值、选中项、searchValue、activeValue（下拉列表中获得焦点的项）。onInternalSelect 有两种调用方式，通过 popup 选项列表点击操作调用、或在多选框中取消选中等。

其他包含输入框展示值（通过 value、options 获取）、选项列表选中值。

### 组件间交互逻辑

#### container

div 容器挂载 onMouseDown、onKeyDown、onKeyUp、onFocus、onBlur 事件。笔者采用如下形式说明该节点的事件处理逻辑。如 container:focus -> inputIcon:focusMock、popup:show; props.onFocus 意味着当 div 容器获得焦点时，inputIcon 按钮也会置为模拟聚焦状态，选项列表弹层显示，同时组件外围绑定的 props.onFocus 将被执行。#if event.which=enter 表示条件语句的主要内容为按键为回车键，antd 实现的条件语句还包含 Select 处于多选模式等。[(#if props.mode=tags) value:change] 表示当 Select 处于 tags 模式时，其值就会被改变。

```javascript
container:focus -> inputIcon:focusMock、popup:show; props.onFocus
container:keydown -> (#if event.which=enter) popup:show; props.onKeyDown
                     (#elseif event.which=backspace) value:change; props.onKeyDown
                     (#else) popup:keydown; props.onKeyDown
container:keyup -> popup:keyup; props.onKeyUp
container:blur -> inputIcon:unfocusMock、popup:hide、searchValue:reset、[(#if props.mode=tags) value:change]; props.onBlur
container(popup):mousedown -> selector:focus、inputIcon:unfocusMock; props.onMouseDown
```

#### selector

selector 有三种交互操作，onToggleOpen、onSearch、onSelect。onToggleOpen 在上文已有交代。onSearch 即上文中的 triggerSearch。onSelect 基于上文中的 onInternalSelect 实现。

#### popup

popup（即 OptionList）有三种交互操作，onToggleOpen、onSelect、onActiveValue、onScroll。onScroll 即 props.onPopupScroll。

### 后记

本文的着力点旨在于分析组件实现的一般方法，最终分析还是不够晓畅、行文还是不够简约。