---
title: hooks 演绎录：useState、useReducer
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: 4fcadd8a
date: 2020-10-24 06:00:00
updated: 2020-10-24 06:00:00
---

为了收集函数式组件内部创建的钩子，fiber reconciler 使用 renderWithHooks 包装函数式组件 Component，继而在构造虚拟节点树时执行该包装函数，完成钩子，随后通过虚拟节点完成实际 dom 树的首渲或更新，并执行钩子。本文将从伪代码推演 useState、useReducer 的逻辑实现，然后再介绍 react 的内部实现。

### 从伪代码说起

组件实例可以转化成虚拟节点，虚拟节点经由 diff 算法对比后，可以作用于实际 dom 树的更新。如果状态更新的标识打在虚拟节点上，那么程序就可以跳过对这个虚拟节点进行 diff。因此在 react 框架的实现中，组件实例转虚拟节点、虚拟节点转 dom 这两个过程有相互交叉的成分。当我们使用伪代码推演 useState、useReducer 实现时，我们将仅截取组件实例转虚拟节点的过程。

因此，我们使用 vNodes 指代虚拟节点、mount 和 update 函数分别指代 dom 的挂载与更新，并用以下伪代码指代首渲和更新的逻辑：

```js
const mount = (vNodes) => {
  console.log("oops，小恶魔要开始挂载组件了哦");
};
const update = (vNodes) => {
  console.log("oops，小恶魔要开始更新组件了哦");
};
const render = (vNodes, isMounted) => {
    if (!isMounted) mount(virtualDoms);
  else update(virtualDoms);
}
```

在有 Component 渲染函数、props 组件实例属性的前提下，我们将通过伪代码做以下事情：
1. 首先，通过包装函数为其注入 useState 函数，此时 useState 将作为包装函数的内部函数
2. 其次，将 useState 从包装函数中独立出来，即如 react 框架那样封装了 hooks 的内部实现
3. 随后，构建 useReducer 作为状态变更处理的超集，毕竟 useState 适用于简单的状态变更
4. 最后，出于提升渲染性能的考虑，我们会在伪代码中对同一钩子的多次状态变更加以合并

完整的伪代码也可以戳 [这里](https://github.com/Alfred-sg/learn-react-hooks)。

#### simple useState

组件树如函数调用栈，它们都表现为需要深度遍历且向上回溯的树形结构。如果我们使用“栈帧”推想组件实例，很容易将每种状态视为“栈帧”中局部变量。即我们可以在渲染函数之上通过包装函数记录和更新状态。

![image](usestate1.png)

我们使用如下的伪代码实现上图的逻辑。这份伪代码使用 setImmediate 模拟用户交互，也仅考虑单个内部状态：

```js
const renderWithHooks = (Component, props) => {
  let firstRender = true;
  let state;
  const setState = (newState) => {
    state = newState;
    firstRender = false;
    const vNodes = Component(props, useState);
    render(vNodes, true);
  };
  const useState = (initialState) => {
    if (firstRender) state = initialState;
    return [state, setState];
  };
  const vNodes = Component(props, useState);
  render(vNodes, false);
};
let immediateID;
const Component = (props, useState) => {
  const [count, setCount] = useState(0);
  console.log(`oops，计数值已经被小恶魔更新成${count}了哦`);
  if (!immediateID){
    immediateID = setImmediate(() => {
      setCount(count + 1);
    });
  };
};
renderWithHooks(Component, {});
```

当 Component 在包装函数内运行时，父级作用域就可以构造 state、setState 局部变量，并通过 useState 显式传入子级作用域。这样，Component 渲染函数就可以直接调用 useState 了。这个示例的问题也很明显，它只适用于单状态变更。

#### external useState

除了使 useState 适用于多状态变更外，我们还将把 useState 提取为外部函数，以便于调用，同时让它适用于多状态变更。依循组件树如函数调用栈的思路，我们可以模仿执行上下文缓存当前被处理的组件及状态。对于多状态变更，我们像 react 那样使用链表存储 hooks。

```js
import render from './common';
let current = null;
let currentHook = null;// 所有钩子收集后移入 current
const isMountPhase = () => {// 判断是否首渲
  return current == null || current.hook == null;
};
  
const useState = (initialState) => {
  let hook;
  if (isMountPhase()){
    hook = { 
      context: current,// 持有 current，以便 setState 找到对应的渲染函数
      state: initialState, 
    };
    hook.setState = (newState) => {
      hook.state = newState;
      current = hook.context;
      const vNodes = current.Component(current.props);
      render(vNodes);
    };
    if (!currentHook){
      currentHook = hook;// 首节点
      currentHook.next = hook;// 首尾相衔
    } else {
      hook.next = currentHook.next;// 首尾相衔
      currentHook.next = hook;// 尾节点
      currentHook = hook;
    };
  } else {
    hook = current.hook;
    current.hook = current.hook.next;// 下一个节点
  };
  
  return [hook.state, hook.setState];
}
const renderWithHooks = (Component, props) => {
  current = {
    Component,
    props,
    hook: null,
  };
  
  const vNodes = Component(props);
  render(vNodes);
  
  current.hook = currentHook.next;// 首节点
  currentHook = null;
  current = null;
};
let immediateID;
const Component = (props) => {
  const [count, setCount] = useState(0);
  console.log(`oops，计数值已经被小恶魔更新成${count}了哦`);
  const [visible, setVisible] = useState(false);
  console.log(`oops，可见性已经被小恶魔更新成${visible}了哦`);
  if (!immediateID){
    immediateID = setImmediate(() => {
      setCount(count + 1);
      setVisible(!visible);
    });
  };
};
renderWithHooks(Component, {});
```

通过如上伪代码，我们看到，使用链表能保证匿名钩子的有序性，以便于有序访问。这样也要求了不能使用控制语句编写钩子，以免破坏钩子的有序性。

#### simple useReducer

如果我们熟知 redux，那么在 useState 基础上构建 useReducer 也是比较容易的，这里仅贴示代码：

```js
import render from './common';
let current = null;
let currentHook = null;// 所有钩子收集后移入 current
const isMountPhase = () => {// 判断是否首渲
  return current == null || current.hook == null;
};
  
const useReducer = (reducer, initialState) => {
  let hook;
  if (isMountPhase()){
    hook = { 
      context: current,// 持有 current，以便 setState 找到对应的渲染函数
      reducer,
      state: initialState, 
    };
    hook.dispatch = (action) => {
      const newState = reducer(action);
      hook.state = newState;
      current = hook.context;
      const vnodes = current.Component(current.props);
      render(vnodes);
    };
    if (!currentHook){
      currentHook = hook;// 首节点
      currentHook.next = hook;// 首尾相衔
    } else {
      hook.next = currentHook.next;// 首尾相衔
      currentHook.next = hook;// 尾节点
      currentHook = hook;
    };
  } else {
    hook = current.hook;
    current.hook = current.hook.next;// 下一个节点
  };
  
  return [hook.state, hook.dispatch];
}
  
const useState = (initialState) => {
  const [state, dispatch] = useReducer((newState) => {
    return newState;
  }, initialState);
  
  return [state, dispatch];
}
const renderWithHooks = (Component, props) => {
  current = {
    Component,
    props,
    hook: null,
  };
  
  const vNodes = Component(props);
  render(vNodes);
  
  current.hook = currentHook.next;// 首节点
  currentHook = null;
  current = null;
};
let immediateID;
const Component = (props) => {
  const [count, setCount] = useState(0);
  console.log(`oops，计数值已经被小恶魔更新成${count}了哦`);
  const [visible, dispatchVisible] = useReducer((action) => {
    if (action == 'show') return true;
    return false;
  }, false);
  console.log(`oops，可见性已经被小恶魔更新成${visible}了哦`);
  if (!immediateID){
    immediateID = setImmediate(() => {
      setCount(count + 1);
      dispatchVisible(!!visible ? 'hide' : 'show');
    });
  };
};
renderWithHooks(Component, {});
```

#### optimized useReducer

如果使用 setState 多次更新同一个状态，那就会造成多次重绘。为了提升渲染性能，其一，我们可以借助浅比较避免重绘；其二，我们可以在渲染线程工作期间将多个状态合而为一，避免多次重绘。我们仅需知道，react 实现了调度器，它会像动效处理一样，需要等待渲染线程完成作业。在如下的伪代码中，我们将使用简单的 == 判断指代浅比较；使用 hook.queue 链表收集状态更新；使用 setTimeout 模拟等待渲染线程完成作业：

```js
import render from './common';
let current = null;
let currentHook = null;// 所有钩子收集后移入 current
let didReceiveUpdate = false;// 状态更新标识
let timer = null;
const scheduleWork = () => {
  if (!timer) timer = setTimeout(() => {
    clearTimeout(timer);
    timer = null;
    const vNodes = current.Component(current.props);
    if (didReceiveUpdate) render(vNodes);
  }, 2000);
}
const isMountPhase = () => {// 判断是否首渲
  return current == null || current.hook == null;
};
  
const useReducer = (reducer, initialState) => {
  let hook;
  if (isMountPhase()){
    hook = { 
      context: current,// 持有 current，以便 setState 找到对应的渲染函数
      reducer,
      state: initialState, 
    };
    hook.dispatch = (action) => {
      const update = {
        reducer,
        action,
        eagerState: hook.state,
      };
      if (!hook.queue){
        const newState = reducer(action);
        hook.eagerState = newState;
        update.next = update;
        hook.queue = update;
      } else {
        update.next = hook.queue.next;
        hook.queue.next = update;
        hook.queue = update;
      };
      current = hook.context;
      if (hook.queue) scheduleWork();
    };
    if (!currentHook){
      currentHook = hook;// 首节点
      currentHook.next = hook;// 首尾相衔
    } else {
      hook.next = currentHook.next;// 首尾相衔
      currentHook.next = hook;// 尾节点
      currentHook = hook;
    };
  } else {
    hook = current.hook;
    current.hook = current.hook.next;// 下一个节点
    const firstUpdate = hook.queue.next;// 取链表头元素
    let update = firstUpdate;
    let newState = hook.state;
    while(update){
      newState = update.eagerState ? update.eagerState : update.reducer(update.action);
      if (hook.state != newState){// 模拟浅比较
        didReceiveUpdate = true;
        hook.state = newState;
      }
      update = update.next;
      if (update == firstUpdate) break;
    };
  };
  
  return [hook.state, hook.dispatch];
}
  
const useState = (initialState) => {
  const [state, dispatch] = useReducer((newState) => {
    return newState;
  }, initialState);
  
  return [state, dispatch];
}
const renderWithHooks = (Component, props) => {
  current = {
    Component,
    props,
    hook: null,
  };
  
  const vNodes = Component(props);
  render(vNodes);
  
  current.hook = currentHook.next;// 首节点
  currentHook = null;
  current = null;
};
let immediateID;
const Component = (props) => {
  const [count, setCount] = useState(0);
  console.log(`oops，计数值已经被小恶魔更新成${count}了哦`);
  if (!immediateID){
    immediateID = setImmediate(() => {
      setCount(1);
      setCount(2);
      setCount(2);
    });
  };
};
renderWithHooks(Component, {});
```

简单总结一下我们的伪代码实现，它包含以下种种：
* current：即如执行上下文，记录当前被处理的组件实例、收集钩子等
* currentHook：记录当前组件收集的钩子，收集完后并入 current
* didReceiveUpdate：标识状态是否更新
* renderWithHooks：包装函数
* scheduleWork：模拟调度函数，用于驱动重绘

### react 内部机制

在 fiber reconciler 中，我们同样能看到伪代码中已实现的种种：currentlyRenderingFiber（即伪代码中的 current）、currentHook、didReceiveUpdate、renderWithHooks、scheduleUpdateOnFiber（即伪代码中的 scheduleWork）。currentlyRenderingFiber 中，memoizedState 属性即钩子对象链表。

fiber reconciler 分别使用 mountReducer、updateReducer、rerenderReducer 封装 useReducer 在不同渲染阶段的具体功能。这三者，通过 ReactCurrentDispatcher.current 切换（如当 ReactCurrentDispatcher.current 为 HooksDispatcherOnMount 时，useReducer 即 mountReducer）：
* mountReducer：首渲时调用，初始化创建钩子对象及状态
* updateReducer：异步更新时调用，计算归并状态
* rerenderReducer：同步更新时调用，计算归并状态。同步更新指的是渲染函数内直接调用 dispatch 或 setState，在伪代码中没有表现

对于创建和获取钩子对象，fiber reconciler 分别使用：
* mountWorkInProgressHook：创建钩子对象，作为 memoizedState 链表的尾节点
* updateWorkInProgressHook：顺序取出钩子对象，它会校验首渲和更新时的钩子数量

我们使用下图来表述以上种种函数的关系：

![image](usestate2.png)

首次渲染时，renderWithHooks 包装函数会调用 mountReducer 函数，以创建 memoizedState 钩子对象链表。每个链表节点为 hook。对于每个 hook，驱动状态变更的 dispatchAction 函数在首渲后也变得可用了。dispatchAction 函数的外部表现即 setState 或 dispatch。

当交互事件发生时，我们会调用 dispatchAction 更新状态并驱动重绘。在此期间，dispatchAction 首先会构造 update 对象，并添加 hook.queue.pending 链表中；其次，如果 dispatchAction 第一次调用，那么直接变更状态（通过浅比较，只有更新的状态才会驱动重绘）；最后调用 scheduleUpdateOnFiber 驱动重绘。在重绘流程中，renderWithHooks 会调用 updateReducer。updateReducer 首先会将 hook.queue.pending 链表中的 update 对象刷入 hook.baseQueue，其次视 update 的优先级直接更新状态或等待下一轮调度周期中执行。

![image](usestate3.png)

fiber reconciler 实现了基于优先级的调度算法，它允许按不同的优先级执行 dispatchAction。因此，存在这么种情况，有些 update 的优先级较高，有些较低。fiber reconciler 内部基于 update.lane 判断这个更新是否需要到点执行了。重绘流程本身会有一个 renderLanes，表示本次重绘流程的优先级。通过 isSubsetOfLanes(renderLanes, update.lane) 就可以判断当前的 update 是否需要到点执行了。如果到点，update 就会被用于计算最新状态。如果未到点，update 还会塞回 baseQueue 链表，等待下一次调度中处理。如果 baseQueue 链表中同时有高低优先级 update，排在低优先级 update 后的高优先级 update 既会被用于更新状态，又会被塞回 baseQueue 链表。“将优先级 update 塞回 baseQueue 链表”是为了保证状态的一致性。因为这时 baseQueue 将以 baseState 作为变基状态，baseState 取的是既往值，并不包含高优先级 update 的更新。

当我们在 Component 函数内同步调用 dispatchAction 时，那样 dispatchAction 便是在视图更新的同时执行了。这时 fiber reconciler 会将 didScheduleRenderPhaseUpdateDuringThisPass 标识置为真值。在 renderWithHooks 中，当该标识为真值时，将切换使用 rerenderReducer，并循环调用 Component 渲染函数（fiber reconciler 限制了调用的次数）。rerenderReducer 会取出 hook.queue.pending 链表中的 update 更新状态。这时不必判断 update 的优先级，因为它会和重绘流程的优先级保持一致。
以上，我们就简单解释了 react 中的内部实现，它会体现在 Hook 钩子对象的数据结构中：

```js
export type Hook = {|
  memoizedState: any,// 视图状态
  baseState: any,// 变基状态
  baseQueue: Update<any, any> | null,// 唤起执行的状态更新链表
  queue: UpdateQueue<any, any> | null,// 存储 dispatch 调用链表，一个钩子可以调用多次
  next: Hook | null,// 下一个钩子
|};
type UpdateQueue<S, A> = {|
  pending: Update<S, A> | null,// 等待中的状态更新链表
  dispatch: (A => mixed) | null,// dispatch
  lastRenderedReducer: ((S, A) => S) | null,// 历次 useReducer 的参数 reducer
  lastRenderedState: S | null,// 记录状态
|};
type Update<S, A> = {|
  lane: Lane,// 决定了更新的执行时机，
  action: A,
  eagerReducer: ((S, A) => S) | null,// 存储 reducer
  eagerState: S | null,// 存储 dispatchAction 函数中计算后的状态
  next: Update<S, A>,// 下一个更新
  priority?: ReactPriorityLevel,
|};
```