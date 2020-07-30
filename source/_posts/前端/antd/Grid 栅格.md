---
title: Grid 栅格
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: a696865b
date: 2018-11-25 06:00:00
updated: 2018-11-25 06:00:00
---

antd 提供的 24 栅格系统由 Row, Col 组件实现。

### Row

栅格布局组件 Row 用于设定整行 row 内各组 col 的布局风格。当 props.type 为 ‘flex’ 时，意味着整行采用 flex 布局。在 flex 布局的基础上，props.justify 通过 justify-content 样式影响各 col 元素的水平排列方式；props.align 通过 align-items 样式影响各 col 元素的垂直对齐方式。props.gutter 用于设定各 col 元素的间隔，间隔单位可以是 ‘px’ 或 ‘rem’。Row 支持以对象形式配置 props.gutter，如 { xs, sm, md, lg, xl, xxl }。其中，xs 为 576 px 以下屏幕；sm 为 576 - 768 px 尺寸屏幕；md 为 768 - 992 px 尺寸屏幕；lg 为 992 - 1200 px 尺寸屏幕；xl 为 1200 - 1600 px 尺寸屏幕；xxl 为 1600 px 以上屏幕；

在响应式处理上，Row 组件借助 enquire.js 库实现。其场景如以 { md: 8, lg: 16 } 对象配置 props.gutter，那么，当屏幕尺寸小于 768px 时，就需要将各 col 元素的间隔从 16 px 调整为 8px。Row 组件对这一场景的实现机制为，在 componentDidMount 生命周期使用 enquire.js 库绑定监听函数，以使得当屏幕尺寸变更时，state.screen 当前屏幕状态也会相应得到更新；随后在 render 阶段，由 getGutter 方法获得当前屏幕尺寸下的 col 元素间隔 gutter，并借助 create-react-context 库将 gutter 注入到 Col 组件中。以下是该机制实现的源码。

```javascript
componentDidMount() {
  Object.keys(responsiveMap)
    .map((screen: Breakpoint) => enquire.register(responsiveMap[screen], {
        match: () => {
          if (typeof this.props.gutter !== 'object') {
            return;
          }
          this.setState((prevState) => ({
            screens: {
              ...prevState.screens,
              [screen]: true,
            },
          }));
        },
        unmatch: () => {
          if (typeof this.props.gutter !== 'object') {
            return;
          }
          this.setState((prevState) => ({
            screens: {
              ...prevState.screens,
              [screen]: false,
            },
          }));
        },
        // Keep a empty destory to avoid triggering unmatch when unregister
        destroy() {},
      },
    ));
}

getGutter(): number | undefined {
  const { gutter } = this.props;
  if (typeof gutter === 'object') {
    for (let i = 0; i <= responsiveArray.length; i++) {
      const breakpoint: Breakpoint = responsiveArray[i];
      if (this.state.screens[breakpoint] && gutter[breakpoint] !== undefined) {
        return gutter[breakpoint];
      }
    }
  }
  return gutter as number;
}
```

在样式处理方面，整行 row 采用 border-box 盒模型，相对定位，并嵌套使用 .clearfix 样式规则以清除浮动以及 :before, :after 伪类的内容。其余均较为简单，可参见源码。

### Col

Col 组件主要通过样式类以设定 col 元素的大小和偏移情况。如当配置了 props.offset 属性为 4 时，将相应添加 ant-col-xs-offset-4, ant-col-sm-offset-4, ant-col-md-offset-4, ant-col-lg-offset-4, ant-col-xl-offset-4, ant-col-xll-offset-4 样式类。因此，Col 组件的逻辑主要是将 props 转化成样式类，其实现的重点仍在于 less 样式文件。

当然，各 col 元素的间隔 gutter 由 Row 组件传入。以下是其源码实现，RowContext 通过 create-react-context 库创建，即通过 context 属性传递数据，但只有 Col 组件能加以访问。

```javascript
// Row
render() {
  // ...
  return (
    <RowContext.Provider value={{ gutter }}>
      <div {...otherProps} className={classes} style={rowStyle}>
        {children}
      </div>
    </RowContext.Provider>
  );
}

// Col
render() {
  return (
    <RowContext.Consumer>
      {({ gutter }) => {
        let style = others.style;
        if (gutter as number > 0) {
          style = {
            paddingLeft: (gutter as number) / 2,
            paddingRight: (gutter as number) / 2,
            ...style,
          };
        }
        return <div {...others} style={style} className={classes}>{children}</div>;
      }}
    </RowContext.Consumer>
  )
}
```

在 less 文件中，antd 使用自调用的混合函数创建从 .ant-col-1 到 .ant-col-24 的样式规则。以下是相关源码。

```less
// style/theme/default.less
@grid-columns: 24;

// mixin.less
// index 由 1 到 24，构建样式规则
.make-grid-columns() {
  .col(@index) {
    @item: ~".@{ant-prefix}-col-@{index}, .@{ant-prefix}-col-xs-@{index}, .@{ant-prefix}-col-sm-@{index}, .@{ant-prefix}-col-md-@{index}, .@{ant-prefix}-col-lg-@{index}";
    .col((@index + 1), @item);
  }
  .col(@index, @list) when (@index =< @grid-columns) {
    @item: ~".@{ant-prefix}-col-@{index}, .@{ant-prefix}-col-xs-@{index}, .@{ant-prefix}-col-sm-@{index}, .@{ant-prefix}-col-md-@{index}, .@{ant-prefix}-col-lg-@{index}";
    .col((@index + 1), ~"@{list}, @{item}");
  }
  .col(@index, @list) when (@index > @grid-columns) {
    @{list} {
      position: relative;
      // Prevent columns from collapsing when empty
      min-height: 1px;
      padding-left: (@grid-gutter-width / 2);
      padding-right: (@grid-gutter-width / 2);
    }
  }
  .col(1);
}

.float-grid-columns(@class) {
  .col(@index) { // initial
    @item: ~".@{ant-prefix}-col@{class}-@{index}";
    .col((@index + 1), @item);
  }
  .col(@index, @list) when (@index =< @grid-columns) { // general
    @item: ~".@{ant-prefix}-col@{class}-@{index}";
    .col((@index + 1), ~"@{list}, @{item}");
  }
  .col(@index, @list) when (@index > @grid-columns) { // terminal
    @{list} {
      float: left;
      flex: 0 0 auto;
    }
  }
  .col(1); // kickstart it
}

// index 由 24 到 1，构建样式规则
.loop-grid-columns(@index, @class) when (@index > 0) {
  .@{ant-prefix}-col@{class}-@{index} {
    display: block;
    box-sizing: border-box;
    width: percentage((@index / @grid-columns));
  }
  .@{ant-prefix}-col@{class}-push-@{index} {
    left: percentage((@index / @grid-columns));
  }
  .@{ant-prefix}-col@{class}-pull-@{index} {
    right: percentage((@index / @grid-columns));
  }
  .@{ant-prefix}-col@{class}-offset-@{index} {
    margin-left: percentage((@index / @grid-columns));
  }
  .@{ant-prefix}-col@{class}-order-@{index} {
    order: @index;
  }
  .loop-grid-columns((@index - 1), @class);
}

.loop-grid-columns(@index, @class) when (@index = 0) {
  .@{ant-prefix}-col@{class}-@{index} {
    display: none;
  }
  .@{ant-prefix}-col-push-@{index} {
    left: auto;
  }
  .@{ant-prefix}-col-pull-@{index} {
    right: auto;
  }
  .@{ant-prefix}-col@{class}-push-@{index} {
    left: auto;
  }
  .@{ant-prefix}-col@{class}-pull-@{index} {
    right: auto;
  }
  .@{ant-prefix}-col@{class}-offset-@{index} {
    margin-left: 0;
  }
  .@{ant-prefix}-col@{class}-order-@{index} {
    order: 0;
  }
}

.make-grid(@class: ~'') {
  .float-grid-columns(@class);
  .loop-grid-columns(@grid-columns, @class);
}

// index.less
.make-grid-columns();
.make-grid();

.make-grid(-xs);

@media (min-width: @screen-sm-min) {
  .make-grid(-sm);
}

@media (min-width: @screen-md-min) {
  .make-grid(-md);
}

@media (min-width: @screen-lg-min) {
  .make-grid(-lg);
}

@media (min-width: @screen-xl-min) {
  .make-grid(-xl);
}

@media (min-width: @screen-xxl-min) {
  .make-grid(-xxl);
}
```