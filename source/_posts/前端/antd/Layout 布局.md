---
title: Layout 布局
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 453983f1
date: 2018-11-25 06:00:00
updated: 2018-11-25 06:00:00
---

antd 提供 Layout, Header, Sider, Content, Footer 组件用于划定页面布局。实现上，侧边栏 Sider 组件比较特殊，将在第一部分加以介绍；其余组件均将在第一部分加以介绍。

### Layout

布局容器 Layout, 顶部布局 Header, 内容部分 Content, 底部布局 Footer 在实现上大体相当：都是通过 generator(props) = BasicComponent => Adapter 创建高阶组件 Adapter，再由该高阶组件将 props.prefixCls 注入到 BasicComponent 中。

作为 Layout 组件的 BasicComponent，BasicLayout 组件以 state.siders 记录侧边栏的 id，并允许在子组件中使用 context.siderHook 添加或移除侧边栏，其意义是触发 BasicLayout 组件重绘并判断是否需要添加 .ant-layout-has-sider 样式类。作为 Header, Content, Footer 组件的 BasicComponent，Basic 组件只用于拼接 props.prefixCls, props.className，以构成新的样式类。

Header, Sider, Content, Footer 组件只能作为 Layout 的子组件这一特征，由 less 文件约定，即如果当上述组件不作为 Layout 的子组件时，其样式将得不到正常展示。

### Sider

侧边栏 Sider 组件用于设定布局，其内容可由 Menu 组件 绘制。在 componentDidMount, componentWillUnmount 生命周期中，侧边栏组件也将通过 context.siderHook.addSider, context.siderHook.removeSider 方法，以添加或移除 Layout 容器中记录的侧边栏 id。侧边栏组件既可以根据折叠状态切换按钮展开或折叠（通过 props.collapsible 或 props.collapsedWidth = 0 开启）；也可以根据屏幕尺寸响应式展开或折叠（通过 props.breakpoint 属性开启），其实现借助 window.matchMedia 方法。子组件可通过 context.siderCollapsed 获得侧边栏的展开和折叠状态。以下是响应式展开或折叠功能的实现。

```javascript
// matchMedia polyfill for
// https://github.com/WickyNilliams/enquire.js/issues/82
if (typeof window !== 'undefined') {
  const matchMediaPolyfill = (mediaQuery: string) => {
    return {
      media: mediaQuery,
      matches: false,
      addListener() {
      },
      removeListener() {
      },
    };
  };
  window.matchMedia = window.matchMedia || matchMediaPolyfill;
}

class Sider extends React.Component<SiderProps, SiderState> {
  constructor(props: SiderProps) {
    // ...

    let matchMedia;
    if (typeof window !== 'undefined') {
      matchMedia = window.matchMedia;
    }
    if (matchMedia && props.breakpoint && props.breakpoint in dimensionMap) {
      this.mql = matchMedia(`(max-width: ${dimensionMap[props.breakpoint]})`);
    }
  }

  componentDidMount() {
    if (this.mql) {
      this.mql.addListener(this.responsiveHandler);
      this.responsiveHandler(this.mql);
    }
    
    // ...
  }

  responsiveHandler = (mql: MediaQueryListEvent | MediaQueryList) => {
    this.setState({ below: mql.matches });
    const { onBreakpoint } = this.props;
    if (onBreakpoint) {
      onBreakpoint(mql.matches);
    }
    if (this.state.collapsed !== mql.matches) {
      this.setCollapsed(mql.matches, 'responsive');
    }
  }

  setCollapsed = (collapsed: boolean, type: CollapseType) => {
    if (!('collapsed' in this.props)) {
      this.setState({
        collapsed,
      });
    }
    const { onCollapse } = this.props;
    if (onCollapse) {
      onCollapse(collapsed, type);
    }
  }
}
```

侧边栏组件的断点 props.breakpoint 包含五种可能：xs 为 480px；sm 为 576px；md 为 768px；lg 为 992px；xl 为 1200px；xxl 为 1600px。当设置了 props.breakpoint 时，就能实现响应式展开与折叠功能。

当 props.collapsedWidth 设置为 0，折叠状态切换按钮为 bars 类型。默认的折叠状态切换按钮为 left 或 right 类型。此外，可以使用 props.trigger 设置自定义折叠状态切换按钮。

侧边栏的主题样式通过设置样式类实现，如 ant-layout-sider-light。

侧边栏组件使用 [react-lifecycles-compat](https://github.com/reactjs/react-lifecycles-compat) 封装，以在低版本的 react 中书写 static getDerivedStateFromProps, getSnapshotBeforeUpdate 生命周期方法。