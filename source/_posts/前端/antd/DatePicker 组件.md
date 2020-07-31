---
title: DatePicker 组件
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 1a1fde32
date: 2020-02-13 06:00:00
updated: 2020-02-13 06:00:00
---

新版的 [DatePicker](https://github.com/react-component/picker) 基于 react-hooks 制作，让我们一步步揭开它的面纱。

DatePicker 分为 Picker 单值选择器、RangePicker 范围选择器两类。

### 公共模块

除了时间处理模块可切换使用 day.js 或 moment 以外。Picker、RangePicker 复用了以下三个组件：

* PickerTrigger: 抽象了弹层展示逻辑的组件，基于 rc-trigger 制作，可设置 PickerPanel 的样式和对齐位置等。
* PanelContext: 传递给 PickerPanel 的 context 内容包含 operationRef 面板操作的 ref 值、hideHeader 是否隐藏头部、panelRef 存储面板的 dom 元素、hidePrevBtn、hideNextBtn、onDateMouseEnter、onDateMouseLeave、onSelect 拣选日期时间后回调、hideRanges、open 面板展开状态、defaultOpenValue 面板默认值。
* PickerPanel: 作为弹层内容的公共容器，向上对接 Picker、RangePicker，获得 panelContext、rangeContext；向下对接实际面板 DecadePanel、YearPanel、MonthPanel、WeekPanel、DatePanel、DatetimePanel、TimePanel 等组件。

PickerTrigger、PanelContext 不作介绍。以下仅介绍 PickerPanel。

#### PickerPanel

![image](DatePanel.png)

以 DatePanel 为例，面板既可选择时间，又可切换展示模式（切换为年面板或月面板）。因此，PickerPanel 内置了四种状态值：innerMode 用于记录面板当前的模式（由模式获得实际的展示面板）；sourceMode 用于记录面板之前的模式；mergedValue 用于存储面板的选中值；viewDate 用于存储当前面板的展示值。

当 DatePanel 等实际面板调用 props.onPanelChange 时，就会执行 PickerPanel 中的 onInternalPanelChange 方法，切换 innerMode 面板模式，并记录历史值 sourceMode，并将时间值和面板模式传递给外围。

当 DatePanel 等实际面板调用 props.onSelect 时，就会执行 PickerPanel 中的 setViewDate、triggerSelect 方法，同步更新 mergedValue、viewDate，并将时间值传递给外围。

PickerPanel 通过赋值 operationRef.current 属性，允许上游的 Picker、RangePicker 组件间接调用实际面板的 onKeyDown、onClose 方法。

PickerPanel 额外负责绘制面板尾部内容。

#### DatePanel

实际面板仅以 DatePanel 举例说明。

DatePanel 面板中 Header 头部使用公共的组件。通过传递 props.onSuperPrev、props.onPrev、props.onNext、props.onSuperNext 控制 Header 头部是否显示上、下一步等按钮。Header 头部文本通过格式化 viewDate 计算获得。

DatePanel 面板中 DateBody 内容基于 viewDate 计算起始日期，然后步进计算展示单元内容。每个单元都会与对应的日期挂钩，点击时会调用 props.onSelect 方法将日期透出外围。

### Picker

Picker 主要负责组织顶层逻辑、渲染实际的触发器。触发器为 input 输入框，可以包含后缀图标、clearNode 清除图标。

作为顶层容器，Picker 包含四种状态：mergedValue 记录受控模式下的外部传入值以及日期选择器的最终值；selectedValue 记录由 PickerPanel、input、clearNode 引起的动态变更值（表现值），初始值与 mergedValue 等值；text 记录输入框中的展示值，通过 selectedValue 格式化处理后获得；mergedOpen 记录 PickerPanel 的展开折叠状态。

selectedValue 初始值与 mergedValue 等值。当 mergedValue 变更或 mergedOpen 变更为否值时，selectedValue 将被刷新为 mergedValue；PickerPanel、input、clearNode 都将刷新 selectedValue 以及 mergedValue。triggerChange(newValue) 即作为变更 selectedValue、mergedValue 的同一调用接口，它会将选中的时间值以及格式化后的时间值传递给外围。PickerPanel 变更面板表现值时，将调用

triggerOpen(newOpen, preventChangeEvent) 负责变更 mergedOpen。若 newOpen、preventChangeEvent 同时为否值，triggerOpen 内部会调用 triggerChange 更新选中值。

input 与 Picker 的对接较为复杂，需要支持的交互行为包含：鼠标点击时获得焦点并展示 PickerPanel；一般按键时展示 PickerPanel，直到按键结束时调用 triggerChange（Tab 按键行为可传递给 PickerPanel）；失焦时隐藏 PickerPanel，（如果点击发生在输入框外围）调用 setSelectedValue 将 selectedValue 更新为 mergedValue，重新计算输入框展示值。以下为其源码：

```javascript
const [inputProps, { focused, typing }] = usePickerInput({
  blurToCancel: needConfirmButton,
  open: mergedOpen,
  triggerOpen,
  forwardKeyDown,
  isClickOutside: target =>
    !elementsContains(
      [panelDivRef.current, inputDivRef.current],
      target as HTMLElement,
    ),
  onSubmit: () => {
    triggerChange(selectedValue);
    triggerOpen(false, true);
    resetText();
  },
  onCancel: () => {
    triggerOpen(false, true);
    setSelectedValue(mergedValue);
    resetText();
  },
  onFocus,
  onBlur,
});

function usePickerInput({
  open,
  isClickOutside,
  triggerOpen,
  forwardKeyDown,
  blurToCancel,
  onSubmit,
  onCancel,
  onFocus,
  onBlur,
}: {
  open: boolean;
  isClickOutside: (clickElement: EventTarget | null) => boolean;
  triggerOpen: (open: boolean) => void;
  forwardKeyDown: (e: React.KeyboardEvent<HTMLInputElement>) => boolean;
  blurToCancel?: boolean;
  onSubmit: () => void;
  onCancel: () => void;
  onFocus?: React.FocusEventHandler<HTMLInputElement>;
  onBlur?: React.FocusEventHandler<HTMLInputElement>;
}): [
  React.DOMAttributes<HTMLInputElement>,
  { focused: boolean; typing: boolean },
] {
  const [typing, setTyping] = React.useState(false);
  const [focused, setFocused] = React.useState(false);
  const preventBlurRef = React.useRef<boolean>(false);

  const inputProps: React.DOMAttributes<HTMLInputElement> = {
    onMouseDown: () => {
      setTyping(true);
      triggerOpen(true);
    },
    onKeyDown: e => {
      switch (e.which) {
        case KeyCode.ENTER: {
          if (!open) {
            triggerOpen(true);
          } else {
            onSubmit();
            setTyping(true);
          }

          e.preventDefault();
          return;
        }

        case KeyCode.TAB: {
          if (typing && open && !e.shiftKey) {
            setTyping(false);
            e.preventDefault();
          } else if (!typing && open) {
            if (!forwardKeyDown(e) && e.shiftKey) {
              setTyping(true);
              e.preventDefault();
            }
          }
          return;
        }

        case KeyCode.ESC: {
          setTyping(true);
          onCancel();
          return;
        }
      }

      if (!open && ![KeyCode.SHIFT].includes(e.which)) {
        triggerOpen(true);
      } else if (!typing) {
        forwardKeyDown(e);
      }
    },

    onFocus: e => {
      setTyping(true);
      setFocused(true);

      if (onFocus) {
        onFocus(e);
      }
    },

    onBlur: e => {
      if (preventBlurRef.current || !isClickOutside(document.activeElement)) {
        preventBlurRef.current = false;
        return;
      }

      if (blurToCancel) {
        setTimeout(() => {
          if (isClickOutside(document.activeElement)) {
            onCancel();
          }
        }, 0);
      } else {
        triggerOpen(false);
      }
      setFocused(false);

      if (onBlur) {
        onBlur(e);
      }
    },
  };

  React.useEffect(() =>
    addGlobalMouseDownEvent(({ target }: MouseEvent) => {
      if (open) {
        if (!isClickOutside(target)) {
          preventBlurRef.current = true;

          window.setTimeout(() => {
            preventBlurRef.current = false;
          }, 0);
        } else if (!focused) {
          triggerOpen(false);
        }
      }
    }),
  );

  return [inputProps, { focused, typing }];
}
```

#### RangePicker

RangePicker 包含以下内部状态：activePickerIndex 当前激活的面板；mergedValue 实际值；selectedValue 选中的时间（可能包含不可选的时间）；mergedDisabled、disabledStartDate、disabledEndDate 不可选时间；rangeHoverValue 选中值范围；hoverRangedValue 鼠标悬浮选中范围；mergedModes 面板模式；mergedOpen 面板展开折叠状态。状态或方法会经由 RangeContext、PanelContext 传递给下游的实际面板 DatePanel 等。此处不作详解。

### 后记

因笔者精力有限，这篇文章仅点到为止，以备查用。个中不足处，还望包涵。