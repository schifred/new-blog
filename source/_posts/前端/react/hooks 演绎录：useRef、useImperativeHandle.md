---
title: hooks 演绎录：useRef、useImperativeHandle
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: 3298a9a
date: 2020-11-09 06:00:00
updated: 2020-11-09 06:00:00
---

### useRef

useRef 有两种功能表现：作为缓存；设置 ref 引用。

```jsx
function AComp(){
  const inputRef = useRef(null);
  
  return (
    <>
      <input ref={inputRef} />
    </>
  );
}
```

通过前面几篇文章的介绍，我们已经了解到，在 renderWithHooks 包装函数的执行过程中，fiber reconciler 会构造 fiber.memoizedState 存储 hook 链表。useRef 也会作为一个 hook 对象存入 fiber.memoizedState 链表中（首渲阶段）。在这个 hook 对象中，hook.memoizedState 属性即 ref 对象（如示例代码中的 inputRef）。随着我们在渲染过程中设置 inputRef.current 的值，在重绘阶段，useRef 将直接读取 hook.memoizedState 属性。

在这套机制下，useRef 能作为缓存已不必多言，那么 useRef 又是怎样设置组件的 ref 引用的呢？以示例代码为例，input 元素带有 ref 属性，即 input 元素对应的 fiber 节点设置了 ref 属性。在 render 阶段处理该 fiber 节点时，fiber.flags 会包含 Ref 标识（通过 makeRef 函数处理）。等到 mutation 子阶段，fiber reconciler 会通过 commitAttachRef、commitDetachRef 绑定和解除引用。在 commitAttachRef 函数执行过程中，若 ref 属性的值为函数，则执行 ref(instance)；若为对象，则设置 ref.current = instance。这样也就解释了 inputRef.current 为什么是 input 节点了。

### useImperativeHandle

useImperativeHandle 接口可用于设置 ref 的内容。在常规为组件设置 ref 引用时，引用的值会被置为组件实例或 host 节点，如上文中的 inputRef.current。在使用 useImperativeHandle 接口后，我们可以基于特定目的裁剪 ref 引用的值。如下示例代码：

```jsx
const Input = React.forwardRef((props, ref) => {
  const inputRef = useRef(null);
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  
  return (
    <>
      <input ref={inputRef} />
    </>
  );
});
function App() {
  const ref = useRef();
  const handleClick = useCallback(() => ref.current.focus(), [ ref ]);
  return (
    <div>
      <Input ref={ref} />
      <button onClick={handleClick}>获取焦点</button>
    </div>
  )
}
```

useImperativeHandle 可以设置三个参数：ref 待设置的 ref 对象；create 函数创建 ref 引用内容；deps 依赖对象。

```js
useImperativeHandle(ref, create, deps);
```

如果我们使用 useImperativeHandle 来理解常规设置 ref 引用的方式，那么，可以认为，在常规方式下，create 函数返回的就是组件的实例 instance 或 host 节点。因此，在实现上，useImperativeHandle 会使 fiber.flags 包含 HookLayout，以便在 render 阶段调用 create 函数去创建 instance（不只是组件实例），然后再执行 ref(instance) 或设置 ref.current = instance。这一过程由 ReactFiberHooks 模块中的 imperativeHandleEffect 函数实现。

### 后记

因为 useRef、useImperativeHandle 的实现较为简单，本文仅点到为止。