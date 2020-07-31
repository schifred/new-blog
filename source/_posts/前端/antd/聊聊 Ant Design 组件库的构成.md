---
title: 聊聊 Ant Design 组件库的构成
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 1a1fde32
date: 2020-02-28 06:00:00
updated: 2020-02-28 06:00:00
---

### 底层功能组件

#### rc-util

KeyCode: 记录了各按键的编码，并提供按键判断工具。

#### hooks

useMergedState(defaultStateValue, option) 主要用于表单项的 value 值状态处理，也可以用于其他状态处理。逻辑上，首先以 option.value、option.defaultValue、defaultStateValue 的顺序设置内部 value 状态值，该 value 值经 option.postState 转变后透出使用；同时透出的有 triggerChange 方法既能改变内部 value 状态值，又能触发 onChange 回调。

```javascript
function useControlledState<T, R = T>(
  defaultStateValue: T | (() => T),
  option?: {
    defaultValue?: T | (() => T);
    value?: T;
    onChange?: (value: T, prevValue: T) => void;
    postState?: (value: T) => T;
  },
): [R, (value: T) => void] {
  const { defaultValue, value, onChange, postState } = option || {};
  const [innerValue, setInnerValue] = React.useState<T>(() => {
    if (value !== undefined) {
      return value;
    }
    if (defaultValue !== undefined) {
      return typeof defaultValue === 'function'
        ? (defaultValue as any)()
        : defaultValue;
    }
    return typeof defaultStateValue === 'function'
      ? (defaultStateValue as any)()
      : defaultStateValue;
  });

  let mergedValue = value !== undefined ? value : innerValue;
  if (postState) {
    mergedValue = postState(mergedValue);
  }

  function triggerChange(newValue: T) {
    setInnerValue(newValue);
    if (mergedValue !== newValue && onChange) {
      onChange(newValue, mergedValue);
    }
  }

  return [(mergedValue as unknown) as R, triggerChange];
}
```

#### rc-align


#### rc-trigger

rc-trigger 作为触发器，它抽象了弹层触发的逻辑。功能点包含：

* 触发节点：children 触发节点。
* 弹层内容：popup 弹层内容；getPopupContainer 获得弹层容器。
* 渲染、销毁时机：forceRender 弹层未显示时予以绘制；destroyPopupOnHide 弹层隐藏时销毁。
* 弹层显示、隐藏时机：action 何种用户行为展示；mouseEnterDelay 鼠标移入后延迟展示；mouseLeaveDelay 鼠标移出时延迟隐藏；popupVisible 受控显示弹层；defaultPopupVisible 弹层默认状况。
* 弹层位置：alignPoint 跟随鼠标位置动态变动；popupAlign 弹层位置；builtinPlacements 触发元素和弹层的对齐关系隐射 { topLeft: { points: [‘tl’, ‘tl’] }} 两元素左上角对齐；popupPlacement 设置对齐位置。
* 弹层样式：popupClassName 样式名；getPopupClassNameFromAlign；popupStyle 样式；prefixCls 样式名前缀。zIndex 弹层 zIndex；stretch 弹层大小随触发元素动态变更。
* 弹层动效：popupTransitionName 弹层动效；maskTransitionName 蒙层动效。
* 蒙层：mask 是否显示蒙层；maskClosable 点击蒙层关闭弹层；getDocument 文档节点，绑定点击隐藏冒牌事件。
* 弹层交互：onPopupVisibleChange 弹层显示时回调；onPopupAlign 弹层位置调整时回调。
* ref 引用：getRootDomNode 获取触发节点的 dom 元素；getPopupDomNode 获取弹层的 dom 元素。

### 通用组件

#### 按钮 Button

* 功能特性：
  * 绘制按钮。
  * 绘制按钮组。
* 样式特性：
  * type 按钮类型：primary 主按钮、默认按钮、dashed 虚线按钮、danger 危险按钮和 link 链接按钮。其中，link 按钮以 a 节点渲染，其他按钮以 button 按钮渲染。
  * disabled、loading 按钮状态：默认正常状态、disabled 不可用状态、loading 加载中状态。加载中状态按钮将额外绘制加载图标。
  * shape 按钮形状：默认小圆角按钮、circle 圆形按钮、round 全圆角按钮。
  * size 按钮尺寸：large 大尺寸按钮、默认中尺寸按钮、small 小尺寸按钮。
  * block 按钮：100% 宽度构成块状按钮。
  * ghost 按钮：背景透明的按钮。
  * icon 图标：按钮内置图标。

按钮内容为双汉字，在双汉字之间插入空格。通过 ConfigProvider 的 autoInsertSpaceInButton 属性可以避免这一行为。

#### 图标 Icon

图标基于 [ant-design-icons](https://github.com/ant-design/ant-design-icons) 制作。

* 功能特性：
  * 绘制内置 svg 图标。
  * component 设置自定义 svg 图标渲染组件，自定义 svg 图标通过 [@svgr/webpack](https://www.npmjs.com/package/@svgr/webpack) 加载，参看 [自定义 SVG 图标](https://ant.design/components/icon-cn/#%E8%87%AA%E5%AE%9A%E4%B9%89-SVG-%E5%9B%BE%E6%A0%87)。
  * tabIndex 限制 tab 按键点选顺序。
  * viewbox 截取 svg 图片，参看 [秒懂 svg 图片的 viewbox](https://www.jianshu.com/p/4422c05ff0f2)。
  * onClick 点击交互等。
  * Icon.createFromIconfontCN 绘制 iconfont 图标，参看 [自定义 font 图标](https://ant.design/components/icon-cn/#%E8%87%AA%E5%AE%9A%E4%B9%89-font-%E5%9B%BE%E6%A0%87)。
  * Icon.setTwoToneColor 设置双色图标的主色。
* 样式特性：
  * theme 图标主题风格：outlined 默认描线、filled 实心、twoTone 双色。
  * type 图标类型：图标类型和 theme 主题风格构成按钮的编码类型，用于查询确切的 svg 图标。
  * spin 旋转图标。
  * rotate 旋转角度：基于 transform 样式制作。
  * twoToneColor 双色图标的主色。
  * className、style 设置 svg 图标颜色等。

#### 排版 Typography

* 功能特性：
  * 绘制 Title 标题、Text 文本、Paragraph 段落。
  * editable 编辑功能。编辑按钮使用 TransButton 组件包裹，便于使用 tab、enter 按键交互以及支持失焦、聚焦事件。编辑按钮点击时将绘制 Textarea。交互行为支持 editable.onStart、editable.onChange。
  * copyable 复制功能。copy 功能借助 [copy-to-clipboard](https://github.com/sudodoki/copy-to-clipboard) 库实现。复制按钮同样使用 TransButton 组件包裹。交互行为支持 copyable.onCopy；copyable.text 可设置复制内容。
  * ellipsis 文本省略，ellipsis.expandable 设置折叠展开功能。对于溢出的文本，ant-design 会借助 raf 在下一帧省略文本内容。文本省略采用两种方案实现：首先尝试使用 css 样式（-webkit-line-clamp、text-overflow 属性）；其次使用两分法查询待显示的文本内容，同时会合并文本节点。ant-design 同时会借助 rc-resize-observer 在屏幕大小改变调整显示文本。交互行为支持 ellipsis.onExpand。
* 样式特性：
  * Title 标题按 level 设置不同级别，分别用 h1、h2、h3、h4 节点渲染。
  * Text 文本使用 span 节点绘制；Paragraph 段落使用 div 节点绘制。
  * type 文字类型：默认主要文字、secondary 次要文字浅灰色、warning 警告文字橘黄色、danger 错误文字红色。disabled 失效文字。mark 使用 mark 节点渲染。code 使用 code 节点渲染。underline 使用 u 节点渲染。delete 使用 del 节点渲染。strong 使用 strong 节点渲染。

### 布局组件

#### 栅格 Grid

* 功能特性：
  * 提供栅格布局功能。栅格布局基于 less 函数制作，并运用了 MediaQuery 样式特性。
* 样式特性：
  * flex 布局：Row 组件默认使用 block 绘制，可以基于 type 属性使用 flex 布局。flex 布局下，Row#justify 属性设置元素的水平对齐方式；Row#align 属性设置元素的垂直对齐方式；Col#order 属性设置元素的展示顺序。
  * Row#gutter 栅格间距（Row 组件设置左右减半负边距，Col 组件设置左右边距），间距可以根据屏幕大小动态调整。屏幕大小通过 enquire.js 库嗅探，以回调形式重绘组件。间距通过 Context 机制传入 Col 组件中。

#### 布局 Layout

* 功能特性：
  * 页面布局，包含 Header 头部、Content 内容、Sider 侧边栏、Footer 尾部。侧边栏可以展开折叠，由 Layout 组件通过收集折叠的侧边栏标识传入子组件中。
* 样式特性：
  * 渲染时，Layout 组件使用 section 节点；Header 使用 header 节点；Content 使用 main 节点；Footer 使用 footer 节点。Layout 采用 flex 布局，主轴为垂直方向。
  * Sider#collapsible 激活侧边栏的展开折叠功能。其一可以通过触发按钮展开折叠侧边栏，其二基于 window.matchMedia 方式响应式展开折叠（触发按钮有两种，Sider 组件的展开折叠按钮、Sider 外围通过 Layout 获取状态的展开折叠按钮）。侧边栏内容由子组件渲染，侧边栏的展开折叠状态通过 Context 机制传递。

Sider#theme 设置侧边栏的样式主题。

### 导航组件

#### 固钉 Affix

* 功能特性：
  * 绘制固钉。固钉会随着 target 节点的事件或者屏幕大小的调整改变位置。固钉的位置调整首先取决于 target 节点和固钉的渲染内容，因此 ant-design 首先会等待渲染完成，借助 raf （封装成方法的装饰器 throttleByAnimationFrameDecorator）调用 setState 重绘，重绘阶段经由 componentDidUpdate 生命周期计算固钉的偏移量等样式，再次使用 setState 进行重绘。依据屏幕大小的动态调整借助于 rc-resize-observer 库；依据 target 节点事件的动态调整借助于 rc-util 库绑定事件。

#### 面包屑 Breadcrumb

Breadcrumb 组件测试脚本通过 MemoryRouter 组件实现。

* 功能特性：
  * 绘制面包屑。面包屑内容既可以使用 Breadcrumb.Item 组件渲染，又可以基于 routes 路由属性渲染（基于路由渲染时，额外可以使用 itemRender 渲染函数获取面包屑元素）。
* 样式特性：
  * separator 分隔符，既可以通过 Breadcrumb#separator 属性设置，又可以通过 Breadcrumb.Separator 组件渲染。
  * Breadcrumb.Item#href 链接，设置后将使用 a 节点渲染，不然使用 span 节点渲染。
  * Breadcrumb.Item#overlay 下拉菜单的内容，下拉菜单通过 DropDown 组件渲染，展示位置在正下方。

#### 下拉菜单 DropDown

* 功能特性
  * 提供下拉菜单的渲染容器。下拉菜单的触发节点 DropDown#children 和下拉菜单 DropDown#overlay 的内容都由开发者提供，一般菜单内容为 Menu 组件渲染内容。DropDown 会为下拉菜单内容传入 selectable=false, focusable=true 以及 expandIcon 图标属性。渲染容器基于 rc-dropdown 库实现，rc-dropdown 内部使用 rc-trigger 库。
  * DropDown.Button 不止于提供下拉菜单的渲染容器，同时也将触发节点设置为按钮。它的实现借助于 Button、DropDown 组件。
* 样式特性：
  * DropDown#getPopupContainer：基于 rc-trigger 库设置下拉菜单的渲染父节点，作为定位节点。
  * DropDown#align：基于 rc-trigger 库设置下拉菜单位置调整依据。
  * DropDown#placement：基于 rc-trigger 库设置下拉菜单的弹出位置。
  * DropDown#transitionName：基于 rc-trigger 库设置下拉菜单展示隐藏时的动效。
  * DropDown#trigger：基于 rc-trigger 库设置触发下拉菜单的行为 click、hover、contextMenu。当由 contextMenu 鼠标右键触发时，基于 rc-trigger 特性菜单位置也会跟随鼠标移动。
  * DropDown#forceRender、DropDown#mouseEnterDelay、DropDown#mouseLeaveDelay 等：均基于 rc-trigger 库实现。

#### 导航菜单 Menu

详情参考 菜单组件源码分析。

* 功能特性：
  * 渲染菜单。菜单的基本要素包含 Menu 菜单容器、SubMenu 子菜单、MenuItem 菜单项。菜单以 context 机制对外承接 Sider 侧边栏组件，对内透传折叠、主题状态到 SubMenu、MenuItem 组件。菜单渲染基于 rc-menu、Tooltip 组件。
  * Menu 菜单容器，组织菜单顶层的展示模式、展开折叠内容及动效。
  * SubMenu 子菜单。
  * MenuItem 菜单项，以 Tooltip 组件绘制。
* 样式特性：
  * Menu#mode 展示模式：vertical 垂直、horizontal 水平、inline 子菜单内嵌三种。在内嵌子菜单模式下，展开的子菜单通过 state.openKeys, state.inlineOpenKeys 交换实现。
  * Menu#theme 主题：light 浅色、dark 深色两种。
  * Menu#openKeys、Menu#defaultOpenKeys 展开的子菜单；Menu#onOpenChange 展开的子菜单变更时调用；Menu#subMenuCloseDelay、Menu#subMenuOpenDelay 子菜单展开、折叠时延；
  * Menu#selectedKeys、Menu#defaultSelectedKeys 选中的菜单项；Menu#onSelect、Menu#onDeSelect 菜单项选中、取消选中时调用；Menu#selectable 是否可选；Menu#multiple 是否多选。
  * Menu#forceSubMenuRender 在子菜单展示之前就渲染进 DOM。
  * Menu#inlineCollapsed 内嵌模式下菜单是否收起；Menu#inlineIndent 内嵌模式下菜单缩进宽度。
  * Menu#onClick 菜单项点击时触发。
  * Menu#overflowedIndicator 菜单折叠时图标。

#### Pagination 分页器

分页器基于 [rc-pagination](https://github.com/react-component/pagination) 渲染。分页器分为非简洁模式和简洁模式两种。非简洁模式有三部分构成：总页数、翻页列表、每页显示条数。简洁模式只有翻页功能，没有总页数、每页显示条数。内置状态包含 current 当前页码、pageSize 每页显示条目数。外置状态 total 总页数用于更新当前页码。翻页功能包含跳转上/下一页；跳转前/后三（或五）页（当前页码在总页码中间态时）。操作方法基于简单改变 current、pageSize 延伸出跳转上/下一页、通过按键改变页码等方法。改变每页显示的 Select 组件由 rc-pagination 通过 props 属性传值。

#### PageHeader 页头

页头组合了 Breadcrumb 面包屑、Avatar 头像、Tag 标签等组件，用于绘制页头。

#### Steps 步骤条

步骤条基于 [rc-steps](https://github.com/react-component/steps) 渲染。Steps 组件主要包含一些样式特性，状态管理逻辑基本没有。当浏览器支持 flex 布局时，Step 采用 flex 布局渲染；不支持时，Step 采用百分比渲染：默认类型的 Steps 会用 margin 值为最后一个元素预留宽度（通过定时器计算最后一个元素的宽度，更新 state 以重绘），导航类型的 Steps 通过百分比均分宽度。

### 数据录入

#### AutoComplete 自动完成

自动完成组件基于 Select、Input 组件制作，主要逻辑也由 Select 组件承担。

#### Checkbox 复选框、Radio 单选框

Checkbox 复选框、Radio 单选框可以参考 Radio, Checkbox。

##### Cascader 级联选择

rc-cascader

级联选择基于 [rc-cascader](https://github.com/react-component/cascader) 制作。rc-cascader 的实现又基于 rc-trigger 抽象弹层逻辑、array-tree-filter 扁平化、过滤树形数据。

实际的触发节点 props.children 通过 React.cloneElement 绑定 onKeyDown 键盘事件。如果 props.children 定义了 onKeyDown 方法，则取用该方法进行处理；如果没有，则使用 rc-cascader 内置的处理逻辑，包含 up/down 上下选、left/backspace 回选、right 向前选、esc/tab 隐藏等功能。

在 rc-cascader 中，array-tree-filter 负责根据级联选择的动态 value 值计算待展示的弹层菜单列表（即弹层菜单中，第二列起的次级列表，其与树结构首层混合，构成当前下拉菜单中展示的完整内容）。菜单中每个选项根据 label 值渲染内容，同时可以包含展开、加载中按钮；选项支持的交互行为包含双击隐藏弹层、点击选中、鼠标移入移出选中。选中的选项存入本地缓存中，在弹层显示状态更新时，用于使选项列表滚动至选中元素上（通过节点的 scrollTop 属性设置）。选项选中逻辑为：如果选中选项为否值或 disabled 状态，直接返回；如果级联选择开启了远程加载 loadData 功能，且选中选项不是叶子节点，首先基于 changeOnSelect 向外透出选中列表，然后更新级联选择的 state.activeValue 状态值并加载远程数据；如果只使用本地数据，且选中选项是叶子节点，向外透出选中列表并更新 state.value 状态值；如果只使用本地数据，根据 expandTrigger 触发行为向外透传选中列表、条件隐藏弹层，并更新 state。value 状态值。

Cascader

Cascader 组件在 rc-cascader 的基础上，默认使用输入框作为触发节点，也可以使用 props.children 替换。输入框有只读态、showSearch 搜索态两种：只读态基于 rc-cascader 点击展开弹层；搜索态阻止点击事件冒泡、支持失焦行为、键盘事件阻止 space/backspace 按键隐藏弹层、change 事件改编 inputValue 值（渲染时更新弹层选项列表）。选中的值在输入框中的表现通过固定定位的 span 节点渲染。输入框内可设置 allowClear 清空图标、suffixIcon 后缀图标。

输入框触发器能基于 SizeContext 获得外层传入的 size 尺寸。Cascader 对外提供的 ref 引用，可以使用 focus、blur 方法获得或失去焦点；input 属性即输入框节点。

全量的选项列表会通过扁平化处理。在渲染时，Cascader 会基于输入框的值进行筛选；筛选内容有最大展示条目限制；筛选内容有最大展示条目限制；选项内容可排序（以上，均可通过 props.showSearch 设置）。当没有（匹配的）选项时，展示空态。

弹层位置可基于 props.popupPlacement 或通过 ConfigContext 传入的 direction 设置（右下或左下对齐）。弹层大小可基于 props.showSearch.matchInputWidth 设置为与输入框等宽。弹层容器可基于 props.getPopupContainer 或通过 ConfigContext 传入的 getPopupContainer 设置。

Cascader 组件的值、弹层显示状态均是受控的；对外回调接口包含 onChange、onPopupVisibleChange。

#### DatePicker 日期选择框

DatePicker 组件基于 rc-picker 制作，它指定 moment 作为时间处理工具，并从上下文中获取 locale、size 属性等，以提供时间选择的功能。

#### Form 表单

可参考 Form。

#### InputNumber 数字输入框

Input 输入框