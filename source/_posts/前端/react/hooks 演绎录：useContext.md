---
title: hooks 演绎录：useContext
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: c09cf7a0
date: 2020-11-08 06:00:00
updated: 2020-11-08 06:00:00
---

我们先看一下 useContext 的使用示例：

```jsx
import React, { createContext, useState, useContext, useEffect } from 'react';
const useUserInfo = () => {
  const [userInfo, setUserInfo] = useState(undefined);
  useEffect(() => {
    return request({ /* fetch from remote */ }).then(res => {
      if (res && res.success) setUserInfo(res.data);
    })
  }, [])
  
  return {
    userInfo
  }
}
const AppContext = createContext({});
const Acomp = () => {
    const { userInfo } = useContext(AppContext);
  
  return (
    <>
      {userInfo.name}
    </>
  )
}
const App = () => {
  const userInfoApis = useUserInfo();
  
  return (
    <AppContext.Provider value={userInfoApis}>
      <Acomp />
    </AppContext.Provider>
  )
}
```

我们已经将 fiber reconciler 对组件树的处理与函数调用栈进行对比，也很容易想到，对上下文对象进行入栈、出栈处理，使之形成一个队列；在每个 fiber 节点处理的过程中，上下文对象队列是确定的，因此就可以从队列中获取上下文对象。所需处理的逻辑还包含，当 userInfoApis 上下文对象变更时，驱动 Acomp 组件重绘。fiber reconciler 对此的实现主要在 ReactFiberContext、ReactFiberStack 模块中，本文将通过以下两部分加以介绍：

* context 入栈、出栈：这一部分意在说明子组件怎么读取到 context 信息
* context 变更重绘：这一部分意在说明 context 变更怎么引起子组件的重绘

### context 入栈、出栈

可以看到，ReactFiberStack 模块实现了栈帧的管理。pop 函数用于出栈；push 函数用于入栈；valueStack 以队列形式存储了数据；fiberStack 仅在开发环境中使用，它以队列形式存储了 fiber 节点。假如一个组件树自上而下分别有 A、B、C 三种数据（B 与 C 平级），在处理 B 数据时，B 数据会被存入 stackCursor.current，A 数据通过 push 函数压入 valueStack 中，即 valueStack 仅存储了历史信息。在处理 C 数据时，stackCursor.current 会通过 pop 函数出栈变为 A 数据，然后将 C 数据存入 stackCursor.current。stackCursor 作为 push、pop 函数的实参传递，valueStack 以变量形式存储。

![image](usecontext1.png)

ReactFiberStack 模块处理有一个特点：即它只能访问最后一个元素。对于 React 中 useContext，无论 Provider 组件在哪一个层级，useContext 都需要能访问到通过 Provider 组件注入的 props.value。显然，ReactFiberStack 模块的实现无法满足。下文将说明这一使用需求在 ReactFiberContext 模块中实现。

在 ReactFiberStack 模块的基础上，ReactFiberContext 首先会创建 StackCursor 实例（即 valueCursor），其次提供了 pushProvider、popProvider 函数。考虑到同一个 Context 的 Provider 组件可以在多处挂载，甚至可以嵌套，子组件通过 useContext 函数访问到的上下文数据实际存储于 providerFiber.type._context 中。按渲染器 renderer 的不同，上下文数据或者被存储在 _context.currentValue 中（dom renderer 等主渲染器），或者被存储在 _context.currentValue2 中（art renderer）。随着 providerFiber.type._context 存储了可被访问的最新数据，ReactFiberStack 模块所提供的栈机制会用于存储历史数据，valueCursor 即上一个 Provider 组件 props.value 传入数据，valueStack 存储其余历史数据。再看一下 pushProvider、popProvider 函数的功能：

* pushProvider(providerFiber, nextValue)：在处理 Provider 挂载、更新的起始阶段（beginWork），将 nextValue 值（Provider 组件的 props.value）存入 providerFiber.type._context；valueCursor 存储上一个 Provider 组件的 props.value；valueCursor 原值（上层 Provider 组件的 props.value）存入 valueStack 队列中
* popProvider(providerFiber)：在处理 Provider 挂载、更新的完成阶段（completeWork），将 providerFiber.type._context 回退为 valueCursor 的值（上一个 Provider 组件的 props.value）；随后将 valueCursor 的值置为 valueStack 队列的尾项（即 valueStack 弹出一个值）

以下为 pushProvider、popProvider 函数的简要功能图示（不区分 _context.currentValue 与 _context.currentValue2，valueCursor 与 valueStack）：

![image](usecontext2.png)

### context 变更重绘

类同于发布订阅机制，子组件先订阅上下文数据，然后由 Context.Provider 发起变更通知，驱动订阅组件重绘。在实现上，子组件会持有订阅信息，因此上下文数据变更时，fiber reconciler 会遍历子树，以找出订阅了上下文数据的组件，然后驱动这些组件重绘。出于性能方面的考虑，子组件未必对所有的上下文数据都感兴趣，fiber reconciler 内部使用 changedBits、observedBits 机制用于订阅特定的变更。下文即将予以说明。

我们先来看一下 ReactFiberContext 模块提供的两个函数：
* readContext(context, observedBits)：此即 useContext 接口的内部实现，订阅 context 信息。因为同一个子组件 childFiber 可以订阅多个上下文数据，订阅的上下文数据会存入 childFiber.dependencies 属性中。observedBits 指对哪部分数据感兴趣，默认值为 MAX_SIGNED_31_BIT_INT，表示对所有数据都感兴趣。context、observedBits 会以对象的形式存入 childFiber.dependencies.firstContext 链表中
* propagateContextChange(workInProgress, context, changedBits, renderLanes)：当上下文数据变更时调用，目的是驱动特定的子组件重绘。因此该函数内部会遍历子树，通过 childFiber.dependencies.firstContext 链表比对子组件是否对当前数据变动感兴趣（context 相同，observedBits 与 changedBits 匹配），如果感兴趣，就更新 childFiber.lanes 标识以表示这个组件需要重绘，同时也会更新 childFiber 祖先节点的 childLanes 标识。在 work-loop 循环中，fiber reconciler 就可以根据 lanes、childLanes 标识做出处理了

我们看到，使用 useContext 就可以设置 observedBits，那么，changedBits 在使用时又该怎么设置呢？createContext 接口可以设置次参 calculateChangedBits，它会作为 context._calculateChangedBits 属性，用于在上下文数据变更时计算 changedBits。如果不设置 calculateChangedBits，changedBits 值即为 MAX_SIGNED_31_BIT_INT。为了提升性能，上下文数据会先进行浅比较，若返回真值，fiber reconciler 不会驱动任何子组件重绘。

![image](usecontext3.png)

以下是一个简单的 changedBits、observedBits 的使用示例：

```jsx
import React, { createContext, useState, useContext, useEffect } from 'react';
const useUserInfo = () => {
  const [name, setName] = useState(undefined);
  const [age, setAge] = useState(undefined);
  
  return {
    name,
    setName,
    age,
    setAge,
  }
}
const NameChangedBits = 0b01;
const AgeChangedBits = 0b10;
const AppContext = createContext({}, (prevValue, nextValue) => {
  let result = 0;
  if (prevValue.name !== nextValue.name) {
    result |= NameChangedBits;
  };
  if (prevValue.age !== nextValue.age) {
    result |= AgeChangedBits;
  };
  return result;
});
const NameComp = () => {
    const { name, setName } = useContext(AppContext, NameChangedBits);
  
  return (
    <>
        <input onChange={event => setName(event.target.value)}/>
      {name}
    </>
  )
}
const AgeComp = () => {
    const { age, setAge } = useContext(AppContext, AgeChangedBits);
  
  return (
    <>
        <input onChange={event => setAge(event.target.value)}/>
      {age}
    </>
  )
}
const App = () => {
  const userInfoApis = useUserInfo();
  
  return (
    <AppContext.Provider value={userInfoApis}>
      <NameComp />
        <AgeComp />
    </AppContext.Provider>
  )
}
```