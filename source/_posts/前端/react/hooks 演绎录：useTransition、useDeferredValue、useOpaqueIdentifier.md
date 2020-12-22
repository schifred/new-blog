---
title: hooks 演绎录：useTransition、useDeferredValue、useOpaqueIdentifier
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: '11393516'
date: 2020-11-14 12:00:00
updated: 2020-11-14 12:00:00
---

### useTransition

useTransition 基于优先级调度算法，它会以 NormalPriority 或更低的优先级执行回调。

```js
const AComp = () => {
  const [inputState, setInputState] = useState(undefined);
  const [startTransition, isPending] = useTransition();
  const handleClick = startTransition((event) => {// 设置 callback
    setInputState(event.target.value);
  });
  return (
    <>
      {/* ... */}
      <input onClick={handleClick} />
            {isPending ? 'pending' : inputState}
    </>
  );
}
```

useTransition 实际执行 callback 时的优先级取自 getCurrentPriorityLevel 函数的返回值，默认为 NormalPriority。currentPriority 优先级可以通过 react-dom 输出的 unstable_runWithPriority 接口修改。

useTransition 内部会使用 useState 维护 isPending 状态。当 startTransition 执行时，它会使用 runWithPriority 将 isPending 状态改为 true，此时的优先级默认为 UserBlockingPriority 或更低（若 currentPriority 小于 UserBlockingPriority，则取 currentPriority）。随后 startTransition 会使用 runWithPriority 将 isPending 状态改为 false，并执行 callback，此时的优先级如上文所说，默认为 NormalPriority 或更高。

### useDeferredValue

基于 useEffect 机制调用 setState 延迟设置值。

### useOpaqueIdentifier

基于 useState、useRef 生成唯一标识。

### 参考

[React Concurrent 模式抢先预览下篇: useTransition 的平行世界](https://juejin.im/post/6844903986420514823)