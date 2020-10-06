---
title: hooks钩子篇
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: df62699e
date: 2020-03-08 06:00:00
updated: 2020-03-08 06:00:00
---

### 前言

本文浅述了 react hooks 的执行机制，但在 expirationTime 优先级处理、suspenseConfig 配置、side-effects 执行机制、useContext、useTransition、useResponder 等方面显得力有不逮，并且未采用图文描述的形式。个中错谬，仍需勘正。

react hooks 的使用可参考 [React Hooks的体系设计之一 - 分层](https://zhuanlan.zhihu.com/p/106665408?from=singlemessage)、react-hooks中的一些懵逼点 等文章。

React Hooks 原理 以探索性的方式推想了 hooks 机制的实现原理；[译] 深入 React Hook 系统的原理 也能为理解 hooks 机制带来帮助。

### 主流程

react hooks 在函数式组件的渲染函数内运行生效，其实现基于使用 renderWithHooks 封装渲染函数。在 renderWithHooks 运行过程中，通过感知组件的渲染阶段 —— mount 挂载、update 更新等，继而通过 ReactCurrentDispatcher 切换 useState 等函数的实现，以此在 mount 阶段构建 hook（包含初始状态、状态更新函数），或者在 update 阶段获取最新状态。

hook 以链表的形式存储在 workInProgressFiber.memoizedState 中。在首次渲染完成后，随着 workInProgressFiber 转化成新的 currentFiber，memoizedState 同样也被存入 currentFiber 中。再次渲染时，workInProgressFiber.memoizedState 会被重置为 null，因此基于 currentFiber.memoizedState 的值就可以区分 mount 挂载、update 更新阶段，以此切换 ReactCurrentDispatcher 实现，如 HooksDispatcherOnMount、HooksDispatcherOnUpdate。

以 useState 说明 hook 处理流程如下：

首次渲染时调用 useState，基于 HooksDispatcherOnMount 走 mountState 处理流程，构建 hook 并获取初始状态，随后消费 state 进行渲染，结束 mount 流程
基于 hook 更新状态，在状态更新逻辑的尾部会调用 scheduleWork 启动 fiber 重绘流程
再次渲染时调用 useState，基于 HooksDispatcherOnUpdate 走 updateState 流程，获取并消费最新 state 进行渲染，结束 update 流程
除了 mount 挂载、update 更新外，ReactCurrentDispatcher 另有两种实现态：为了避免 hook 在渲染函数外部执行，fiber reconciler 提供了 ContextOnlyDispatcher 实现；为了实现渲染函数内部当即更新 state，fiber reconciler 提供了 HooksDispatcherOnRerender 实现。逻辑上，渲染函数内部更新 state 使用 do-while 循环更新状态，这时 useState 调用将走 rerenderState 流程，且不会调用 scheduleWork（因为本身就在一个渲染流程中）；fiber reconciler 也限制了这种 state 更新方式的最大次数。

这里贴出 renderWithHooks 的删减版源码：

```js
function renderWithHooks(
  current: Fiber | null,// current fiber
  workInProgress: Fiber,// work-in-progress fiber
  Component: any,// 函数组件的渲染函数
  props: any,// 渲染函数的参数 props
  secondArg: any,// 渲染函数的次参
  nextRenderExpirationTime: ExpirationTime,
): any {
  renderExpirationTime = nextRenderExpirationTime;
  // 缓存当前处理中的 fiber。renderWithHooks 结束置为 null
  currentlyRenderingFiber = workInProgress;

  // 函数体每次执行时重置 workInProgress.memoizedState，借以与 currentFiber.memoizedState 有差别
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.expirationTime = NoWork;

  // 使用 currentFiber.memoizedState 区分 mount、update 阶段，使用不同的 ReactCurrentDispatcher 实现
  // mount 阶段，HooksDispatcherOnMount 将促使 useState 调用走 mountState 流程
  // update 阶段，HooksDispatcherOnUpdate 将促使 useState 调用走 updateState 流程
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount
      : HooksDispatcherOnUpdate;

  // 调用渲染函数，获取 hook 或最新状态，渲染内容
  let children = Component(props, secondArg);

  // 在渲染函数体内更新 state，限制函数体内最大更新次数
  if (workInProgress.expirationTime === renderExpirationTime) {
    let numberOfReRenders: number = 0;

    // do-while 循环获取最新 state
    do {
      workInProgress.expirationTime = NoWork;

      invariant(
        numberOfReRenders < RE_RENDER_LIMIT,
        'Too many re-renders. React limits the number of renders to prevent ' +
          'an infinite loop.',
      );

      numberOfReRenders += 1;

      // Start over from the beginning of the list
      currentHook = null;
      workInProgressHook = null;

      workInProgress.updateQueue = null;

      // HooksDispatcherOnRerender 将促使 useState 调用走 rerenderState 流程
      ReactCurrentDispatcher.current = __DEV__
        ? HooksDispatcherOnRerenderInDEV
        : HooksDispatcherOnRerender;

      // 调用渲染函数，获取最新状态，渲染内容
      children = Component(props, secondArg);
    } while (workInProgress.expirationTime === renderExpirationTime);
  }

  // 将 ReactCurrentDispatcher 置为 ContextOnlyDispatcher，避免 hook 在函数体外调用
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  // didRenderTooFewHooks 为真，意为有些 hook 未执行完全，过早返回 state
  const didRenderTooFewHooks =
    currentHook !== null && currentHook.next !== null;

  renderExpirationTime = NoWork;
  currentlyRenderingFiber = (null: any);

  currentHook = null;
  workInProgressHook = null;

  didScheduleRenderPhaseUpdate = false;

  invariant(
    !didRenderTooFewHooks,
    'Rendered fewer hooks than expected. This may be caused by an accidental ' +
      'early return statement.',
  );

  return children;
}
```

我们可以看到，renderWithHooks 执行过程中处理的缓存变量：

ReactCurrentDispatcher：hook 处理策略，其实现有 HooksDispatcherOnMount、HooksDispatcherOnUpdate、HooksDispatcherOnRerender、ContextOnlyDispatcher，上文已有描述。
currentlyRenderingFiber：缓存当前处理中的 fiber。renderWithHooks 执行中置为 workInProgressFiber，执行完置为 null。这样切分后的功能函数不必以传参的方式获取当前处理中的 fiber。
workInProgressHook：当前处理中的 hook。mount 阶段，通过 mountWorkInProgressHook 构造 workInProgressHook，并存入 workInProgressFiber.memoizedState；update 阶段，通过 updateWorkInProgressHook 从 workInProgressFiber.memoizedState 中获取或拷贝 currentHook 得来。
currentHook：其意义在于 update 阶段起始，workInProgressFiber.memoizedState 会被重置，currentFiber.memoizedState 保存了 hook 的完整链表，因此需要通过 currentHook 从 currentFiber.memoizedState 取出 hook 钩子，拷贝给 workInProgressHook，才有真正要处理的 hook。
renderExpirationTime：组件渲染的 deadline 时间，决定了更新是否需要被应用、还是被挂起。
didScheduleRenderPhaseUpdate：是否在 render 阶段更新 state。renderWithHooks 执行完置为 false，状态更新时判断触发更新的 fiber 是否与 currentlyRenderingFiber 等值，如等值，置为 true。当报错时，fiber reconciler 将基于 didScheduleRenderPhaseUpdate 丢弃各 hook 的 queue.pending，参考文档。
state 类钩子
本节将 useState、useReducer 置为同类，因为 useState 期间所使用的各方法筑基在 useReducer 各方法之上。

hooks 中的状态变更与 redux 相仿，即采用 reducer = (prevState, action) => newState 计算 nextState。通常而言，新的状态计算产生后，fiber reconciler 会调用 scheduleWork 重新调度当前 fiber 的渲染任务。这是 state 变更的主流程。依循上文，这两种钩子的整体处理流程为：

mount 阶段，通过 mountReducer、mountState 创建 hook
基于 hook.dispatch 更新状态
update 阶段，通过 updateReducer、updateState 获取最新的 state，其内部会调用 scheduleWork 重绘 fiber
特别的，若渲染函数内更新状态，通过 rerenderReducer、rerenderState 获取最新的 state，其内部不会触发 fiber 重绘
hook 为链表。其中，hook.baseState 属性用于存储更新前的 state；memoizedState 属性用于存储更新后的 state；queue 属性即一个 UpdateQueue 实例，包含状态变更逻辑、排队中的状态变更规则；baseQueue 就绪中的状态变更规则。baseQueue 按照优先级处理，比如有两组优先级更新，优先级高的会将 memoizedState 置为最新状态，优先级低的仍旧使用 baseState 作为初始状态。

hook.queue 作为一个 UpdateQueue 实例，其包含：lastRenderedState 属性存储上一个 state；lastRenderedReducer 属性存储 reducer 内部状态变更逻辑；dispatch 属性存储用户侧调用的状态变更接口；pending 属性存储排队中的状态变更规则。queue.dispatch 构建时会与 currentlyRenderingFiber 绑定，惟其如此，才能判断状态更新是否在当前 fiber 的 render 阶段，还是异步触发，以此将 didScheduleRenderPhaseUpdate 置为 true。

hook.baseQueue、hook.queue.pending 均为 Update 链表形式。update 包含：eagerReducer 缓存上一个 reducer；eagerState 缓存上一个状态，以便复用；action 状态变更的规则；expirationTime 过期时间，即 deadline 时间，未到该时间点，更新将不予执行；suspenseConfig 意为 suspense 配置。当使用 useReducer 时，reducer 在历次渲染函数中是可变的，如果 reducer 没有发生变更，fiber reconciler 将复用 update.eagerState。

useState、useReducer 中的 hook.dispatch 均基于 dispatchAction 构建，且在 fiber reconciler 内部就会装填参数 fiber、queue，用户侧只需传入 action。hook.dispatch 对外透出时作为 useState、useReducer 返回值中的第二个数组项，调回该方法即可更新状态。其处理流程为：

基于 action 构建 update 实例，并将其添加到 hook.queue.pending 队列中
若在 render 阶段调用 hook.dispatch，更新 expirationTime，等待调度作业阶段获取 state；若否且 fiber 更新为同步任务，调用 reducer 计算最新状态
调用 scheduleWork 调度作业，执行重绘流程
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
function dispatchAction<S, A>(
  fiber: Fiber,// 状态变更所作用的 fiber，用于推断是否在 render 阶段应用更新
  queue: UpdateQueue<S, A>,// hook.queue，用于获取状态变更逻辑
  action: A,// 用户侧传入的状态变更规则 action
) {
  const currentTime = requestCurrentTimeForUpdate();
  const suspenseConfig = requestCurrentSuspenseConfig();
  const expirationTime = computeExpirationForFiber(
    currentTime,
    fiber,
    suspenseConfig,
  );

  // 构建 update 实例
  const update: Update<S, A> = {
    expirationTime,
    suspenseConfig,
    action,
    eagerReducer: null,
    eagerState: null,
    next: (null: any),
  };

  // 将 update 添加到 hook.queue.pending 中
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;

  const alternate = fiber.alternate;

  // 推断是否在 render 阶段应用更新（调用 hook.dispatch）
  if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
  ) {
    // This is a render phase update. Stash it in a lazily-created map of
    // queue -> linked list of updates. After this render pass, we'll restart
    // and apply the stashed updates on top of the work-in-progress hook.
    didScheduleRenderPhaseUpdate = true;
    update.expirationTime = renderExpirationTime;
    currentlyRenderingFiber.expirationTime = renderExpirationTime;
  } else {
    // fiber.expirationTime 为 NoWork 时，采用同步更新处理
    if (
      fiber.expirationTime === NoWork &&
      (alternate === null || alternate.expirationTime === NoWork)
    ) {
      // 应用 queue.lastRenderedReducer 变更状态，并将变更结果存入 update
      // 若状态未变更，直接退出
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        let prevDispatcher;
        try {
          const currentState: S = (queue.lastRenderedState: any);
          const eagerState = lastRenderedReducer(currentState, action);

          // reducer 调用后才会赋值 update.eagerReducer、update.eagerState
          update.eagerReducer = lastRenderedReducer;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            return;
          }
        } catch (error) {
          // Suppress the error. It will throw again in the render phase.
        }
      }
    }
    
    // 调度 fiber 的渲染任务
    scheduleWork(fiber, expirationTime);
  }
}
因为 hook.dispatch 会调度一些异步更新，对于这些异步更新，state 变更在 updateReducer 函数中处理。updateReducer 的处理流程为：

将 hook.queue.pend 排队更新任务队列添加到 hook.baseQueue 就绪更新任务队列中
执行 hook.baseQueue 中优先级足够的更新或复用 hook.dispatch 获得的更新状态
优先级不够的 update 任务等待下次调度作业处理
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: I => S,
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  invariant(
    queue !== null,
    'Should have a queue. This is likely a bug in React. Please file an issue.',
  );

  queue.lastRenderedReducer = reducer;

  const current: Hook = (currentHook: any);

  // fiber 渲染后，尚未执行完成 update 更新，存于 currentFiber
  let baseQueue = current.baseQueue;
  // 排队中的 update 更新，存于 workInProgressFiber
  let pendingQueue = queue.pending;

  // 将排队中的 update 更新添加到 baseQueue
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      let baseFirst = baseQueue.next;
      let pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;// pending 队列尾节点添加 baseFirst
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  if (baseQueue !== null) {
    let first = baseQueue.next;
    let newState = current.baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    do {
      const updateExpirationTime = update.expirationTime;
      // update 优先级不足，将其存入 newBaseQueueList 等待更新
      if (updateExpirationTime < renderExpirationTime) {
        const clone: Update<S, A> = {
          expirationTime: update.expirationTime,
          suspenseConfig: update.suspenseConfig,
          action: update.action,
          eagerReducer: update.eagerReducer,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        // Update the remaining priority in the queue.
        if (updateExpirationTime > currentlyRenderingFiber.expirationTime) {
          currentlyRenderingFiber.expirationTime = updateExpirationTime;
          markUnprocessedUpdateTime(updateExpirationTime);
        }
      } else {
        // This update does have sufficient priority.
        // 同 UpdateQueue，当优先级高的 update 任务在优先级低的 update 任务后
        // 添加到 newBaseQueueList 中，更新处理会执行多次
        if (newBaseQueueLast !== null) {
          const clone: Update<S, A> = {
            expirationTime: Sync, // This update is going to be committed so we never want uncommit it.
            suspenseConfig: update.suspenseConfig,
            action: update.action,
            eagerReducer: update.eagerReducer,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // Mark the event time of this update as relevant to this render pass.
        // TODO: This should ideally use the true event time of this update rather than
        // its priority which is a derived and not reverseable value.
        // TODO: We should skip this update if it was already committed but currently
        // we have no way of detecting the difference between a committed and suspended
        // update here.
        markRenderEventTimeAndConfig(
          updateExpirationTime,
          update.suspenseConfig,
        );

        // hook.dispatch 变更状态后再执行组件的渲染函数
        // 若 reducer 未变更，复用之前的状态 update.eagerState
        // 若 reducer 变更，使用 reducer 计算最新的状态
        if (update.eagerReducer === reducer) {
          newState = ((update.eagerState: any): S);
        } else {
          const action = update.action;
          newState = reducer(newState, action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }

    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }

    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;

    queue.lastRenderedState = newState;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
effect 类钩子
本节将 useEffect、useLayoutEffect、useImperativeHandle、useDeferredValue 置为同类。它们都基于 Effect 数据结构。其包含：tag 指 side-effects 处理类型；create 指 side-effects 处理函数，在 commit 阶段执行，其返回值构成 effect.destroy（组件销毁时执行）；deps 指 side-effects 处理依赖。无论在 mount 还是 update 阶段，effect 类钩子的表现都是将 lastEffect 存入 workInProgressFiber.updateQueue，等待在 commit 阶段被执行。

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
function mountEffectImpl(fiberEffectTag, hookEffectTag, create, deps): void {
  const hook = mountWorkInProgressHook();// 构建 hook
  const nextDeps = deps === undefined ? null : deps;
  currentlyRenderingFiber.effectTag |= fiberEffectTag;
  // pushEffect 构建 effect，并将其以 { lastEffect } 形式存入 workInProgressFiber.updateQueue 中
  hook.memoizedState = pushEffect(
    HookHasEffect | hookEffectTag,
    create,
    undefined,
    nextDeps,
  );
}

function updateEffectImpl(fiberEffectTag, hookEffectTag, create, deps): void {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;

  // 复用之前的 destory
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        pushEffect(hookEffectTag, create, destroy, nextDeps);
        return;
      }
    }
  }

  currentlyRenderingFiber.effectTag |= fiberEffectTag;

  hook.memoizedState = pushEffect(
    HookHasEffect | hookEffectTag,
    create,
    destroy,
    nextDeps,
  );
}
useEffect、useLayoutEffect 的差别在于执行时机。useEffect 将 hookEffectTag 置为 HookPassive，意为等组件渲染完成后触发调用；useLayoutEffect 将 hookEffectTag 置为 HookLayout，它在计算节点样式后、页面刷新前执行。

useImperativeHandle 须结合 forwardRef 一起使用，用于设置父组件的 ref 引用。

其他
useRef 用于缓存数据，特殊之处在于缓存 ref 引用。
useMemo 通过函数计算缓存数据，每次 deps 变更时会重新计算。
useCallback 用于缓存函数，每次 deps 变更时会重新计算。
useContext 读取祖先组件构建的上下文。
useTransition 用于延迟处理，参看 React useTransition。内部处理结合 runWithPriority、ReactCurrentBatchConfig.suspense 机制。
useDeferredValue 用于延迟获取数据（引用形式更新）。内部处理使用 ReactCurrentBatchConfig.suspense 机制
useResponder 响应式处理。

