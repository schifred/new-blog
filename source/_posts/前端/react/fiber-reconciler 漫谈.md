---
title: fiber-reconciler 漫谈
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: 7c21db8b
date: 2020-10-18 06:00:00
updated: 2020-10-18 06:00:00
---

### 前世今生

自 react16 起，react 底层就使用 fiber reconciler 协调器替换 stack reconciler。为什么呢？当 stack reconciler 处理大状态更新时，一方面需要复杂的计算，另一方面需要深度遍历组件树，因此长期工作的 js 执行引擎就会导致渲染线程被挂起，页面出现掉帧。此时，我们可以像处理远程搜索框那样使用防抖函数归并状态；也可以使用 shallowEqual 或虚拟列表等优化组件树渲染。还有其他方法吗？

假想有一个搜索列表，如果能把搜索框和长列表分开来渲染，我们就可以异步调度这两个渲染流程，即时重绘搜索框，延时重绘长列表，实现两者的并发。在实现虚拟 dom 的情况下，一整个渲染流程包含“基于状态更新虚拟节点”、“将更新后的虚拟节点应用于渲染”两个阶段。react 将前者称为 render 阶段，即渲染虚拟节点；后者称为 commit 阶段，即提交虚拟节点，完成 dom 树的渲染等。为了保证视图的一致性，commit 阶段是不能被打断的；render 阶段却可以增量执行。所谓的增量执行，指的是，render 阶段一旦执行过长，它就有必要被挂起，使渲染线程能正常工作；等到渲染完成后，回头再执行 render 阶段操作。这就是任务的并发和挂起，stack reconciler 没法支持。

为了实现任务的增量执行且使视图保持流畅，首先就需要将 stack reconciler 中的同步递归大任务拆解成足够小的任务单元；其次就需要在任务单元切换的间隙，校验是否有渲染作业或渲染线程长期没法工作（无法保证浏览器页面动效的帧率）——如果是，就停用 js 执行引擎，使渲染线程展开工作。在 react 中，作为任务单元的就是 fiber 虚拟节点。上文已有说明，render 阶段可以增量处理 fiber 节点，commit 阶段仍需要批量处理 fiber 树。这是任务挂起的必要实现，也是 js 执行引擎和渲染线程间的调度策略——即按时间分片调度，js 执行引擎只能在给定时间分片内运行，一旦执行过长，就交还给渲染线程工作；如果 js 执行引擎工作期间有一个渲染作业，那也会促使它把主线程。

当有多个渲染流程时，就需要实现 js 执行引擎内多个任务的调度。在交互式环境下，多任务可以按优先级调度，交互要求高的任务，其优先级也高。借助于优先级调度，我们也可以在某个任务单元执行完成后，让后添加的高优先级任务先执行。为了避免低优先级的任务得不到执行（即饿死），优先级会转化成到期时长 timeout，即希望任务被添加后，等到 timeout 时间点时执行。任务被添加时间点 currentTime + timeout 构成任务的 expirationTime 到期时间点，任务会根据到期时间点进行排序，这样就能保证先添加的低优先级任务仍能正常得到执行。这样就能实现多个渲染流程在异步调度模式下的伪并发。

![image](reconciler1.png)

概而论之，fiber reconciler 主要实现了如下特性：
* 允许将可拆分的任务拆解为任务单元，由任务单元构成的任务可以被暂停、复用或取消，以便实现如虚拟节点的增量更新
* 允许为任务设置优先级，任务可并发执行，如在低优先级任务中间隙穿插执行高优先级任务，延迟处理低优先级任务不影响体验

### 整体架构

![image](reconciler2.png)

fiber reconciler 协调器依赖于 renderer 渲染器和 scheduler 调度器。它以 ReactElement 元素树为输入，然后构建 fiber 虚拟节点树并据此完成渲染。

#### fiber

fiber 往前对接 ReactElement，往后作为虚拟节点用于渲染。fiber 作为任务单元，它像栈帧，会持有一些上下文信息。组件树的处理与函数调用栈如此相似：函数内可能会有多个子函数，子函数又会回溯到父函数上，子函数执行前后也可以挂载钩子。因此循着函数调用栈的处理思路，要而言之，fiber reconciler 会在 render 阶段深度遍历 fiber 节点，完成节点的状态更新并收集钩子，获得 fiber 树；在 commit 阶段再深度遍历这棵 fiber 树，执行怪载在 fiber 节点上的副作用钩子。这些副作用钩子包含插入、更新、删除等渲染操作。

fiber reconciler 在 render 阶段处理的 fiber 树称为 work-in-progress tree。这棵树在 render 阶段处理完后也称为 finished-work tree。等到 commit 阶段完成渲染后，finished-work tree 又会存为 current tree。在具体实现上，current fiber、work-in-progress fiber 以 alternate 属性持有对方。这被称为双缓冲技术，以便在状态更新时使用  current fiber 及其树形关系，快速生成 work-in-progress fiber，这样可以节省内存分配和 GC 时间。在双缓冲机制下，我们可以通过 current fiber 是否为 null 值推断当前是首渲，还是更新。首渲为 null，更新非 null。

fiber 以 tag 属性定义逻辑处理策略，值包含 FunctionComponent、MemoComponent、Mode、Block 等。下文仅说明 FunctionComponent 函数式组件的处理策略。

上文提到，fiber 任务单元的优先级由 currentPriority 影响。我们也可以通过 fiber.mode 属性将当前节点设置成同步或并发渲染模式，通常设置 mode 属性，需要该节点作为根节点才行。

fiber 还包含以下属性：

```js
type Fiber = {|
  tag: WorkTag,// fiber 类型，影响内部逻辑，包含 FunctionComponent 等
  elementType: any,// react 元素类型
  type: any,// 组件构造函数
              
  pendingProps: any,// 待使用的 props
  memoizedProps: any,// 已使用的 props
  updateQueue: mixed,// 状态更新或回调链表
  memoizedState: any,// 状态
  stateNode: any,// host 元素
  return: Fiber | null,// 类同返回栈帧，通常是父节点
  child: Fiber | null,
  sibling: Fiber | null,
  index: number,
  flags: Flags,// fiber 节点包含的副作用标识
  subtreeFlags: Flags,// 子树包含的副作用标识，避免深度遍历
  deletions: Array<Fiber> | null,// 删除的节点，用于执行 unmount 钩子
  lanes: Lanes,// 优先级相关，用于计算过期时间
  childLanes: Lanes,
  alternate: Fiber | null,// current fiber、work-in-progress fiber 相互持有对方
|};
```

### 单渲染流程

我们使用下图推演 render、commit 两阶段的逻辑处理流程：


![image](reconciler3.png)

ReactElement 包含 Component 组件、props 属性，两者即可构成 fiber.type、fiber.pendingProps，以此构建 fiber 节点。之后，我们就可以使用包装函数执行 Component(props)。在这个过程中，我们可以：

1. 构建 setState 状态更新逻辑，使得外部调用 setState 可驱动重绘。渲染流程有两个入口，通过根节点挂载，或者通过 setState 更新状态
2. 获得最新状态
3. 获取 context 上下文信息
4. 收集副作用钩子并更新 fiber.flags 标识。fiber.flags 标识说明了这个节点需要执行哪些副作用钩子
5. 获得子节点 fiber.child，并为子节点添加增删改标识（这个标识也由 fiber.flags 实现）
6. 递归处理 fiber.child，处理完后回溯到当前节点，更新 fiber.subtreeFlags、fiber.childLanes 标识。这样可以通过 fiber.subtreeFlags 快速判断子树是否有副作用钩子，不需要深度遍历

这时，我们获得了 render 阶段的最终产物——finished-work tree。在 commit 阶段，通过判断 fiber.flags 标识，程序就可以执行相应的钩子。对于每种类型的钩子，程序还需要根据 fiber.subtreeFlags 标识递归处理子树。

在实际处理上，fiber reconciler 使用 workInProgress 标记当前处理的 fiber 节点，并通过以下两类函数实现 render、commit 两阶段操作。理论上，将这两类函数串起来就可以发起一次渲染流程。
* workLoopSync、workLoopConcurrent：render 阶段通过深度遍历构建 fiber 节点树，并收集钩子
* commitRoot：commit 阶段使用 fiber 节点树执行钩子（包含渲染操作）

#### 钩子类型

fiber reconciler 包含四大类钩子（后三类也构成 commit 阶段的子阶段）：
* passive：下文介绍
* beforeMutation：渲染前，会调用 getSnapshotBeforeUpdate 生命周期函数
* mutation：渲染中，包含 placement 插入、update 更新、deletion 删除等子类钩子，完成实际的渲染作业
* layout：渲染后，会调用 useLayoutEffect 添加的钩子以及 componentDidMount、componentDidUpdate 生命周期函数

passive 类钩子即通过 useEffect 添加的钩子，它比较特殊。passive 语义上是“消极的”，指的是不必即时执行，因此会采用较低的优先级（NormalPriotrity）进行调度。正是因为如此，就可能出现一个渲染流程的 passive 类钩子还未执行完成，另一个渲染流程又唤起执行的情况，所以 flushPassiveEffects 函数既会在 beforeMutiton 子阶段唤起，又会在调度渲染流程前唤起。

部分钩子会在多阶段执行，如 deletion 类钩子既会在 beforeMutation 阶段处理（触发 beforeblur 事件），又会在 mutation 阶段处理（执行 componentWillUnmount 生命周期函数）。四大类钩子也表现为复合钩子，比如 beforeMutation 即包含 passive、snapshot、deletion、placement。

fiber.flags 标识使用了位运算处理，它可以包含 MountPassiveDevEffect。因为 1111 与 0001 按位或计算仍为 1111，我们就说 1111 包含 0001。fiber.flags 包含 MountPassiveDevEffect 也是如此。“fiber.flags 包含 MountPassiveDevEffect”有什么具体意义呢？其意义在于，在组件挂载时，就需要执行相应的 effect 钩子。当 fiber.flags 等于 NoFlags，则意味着节点不必执行任何钩子。

#### render 阶段

这一小节，我们主要介绍 work-loop（指代 workLoopSync、workLoopConcurrent）的工作内容。
workLoopSync、workLoopConcurrent 会使用 while 循环调用 performUnitOfWork。不同的是，workLoopConcurrent 会在 fiber 切换时调用 shouldYied 主动挂起，使渲染线程工作顺畅；workLoopSync 会同步批量处理 fiber 节点，不会被挂起。work-loop 包含两层循环，首先深度向下遍历子节点，完成 Component 调用，直到处理完最后一个叶子节点；其次循环向上回溯父节点，更新父节点的 fiber.subtreeFlags 标识，直到回溯到根节点。

![image](reconciler4.png)

* performUnitOfWork：组织 beginWork、completeUnitOfWork 逻辑。当 beginWork 有返回时，持续调用 beginWork 处理后续子节点或兄弟；无返回时，调用 completeUnitOfWork
* beginWork：执行渲染函数，更新 fiber 的 props、state 等属性，收集钩子；处理数组等形式的子节点；返回 fiber.child。对于 Conext.Provider 等组件，beginWork 还包含 context 入栈等操作
* completeUnitOfWork：循环调用 completeWork。若 completeWork 有返回值；若无返回值，则使用 fiber.sibling 或 fiber.return 进行循环
* completeWork：更新 fiber 的 subtreeFlags、childLanes 等属性。对于 Conext.Provider 等组件，completeWork 还包含 context 出栈等操作（当 work-loop 处理失败时，fiber-reconciler 会使用 unwind 函数完成 context 出栈操作）

在 renderWithHooks 包装函数执行期间，它会调用 Component 函数式组件本身，完成状态的更新、钩子的收集。如渲染函数内调用了 useEffect，它会转化成 effect 对象，存入 fiber.updateQueue 队列，且使 fiber.flags 包含 MountPassiveDevEffect、PassiveEffect、PassiveStaticEffect。这样就能在 commit 阶段通过判断 fiber.flags 标识是否有 passive 类钩子，如果有，就取出 updateQueue 队列中的 effect 对象并处理。 

对于 Component 的返回值 fiber.child，fiber-reconciler 会使用 reconcileChildren 加以处理。在 reconcileChildren 处理期间，fiber-reconciler 会使子节点的 flags 标识包含 Placement、Update、PlacementAndUpdate 等，表示该子节点有插入、或更新、或插入并更新操作。
对于类组件，render 阶段还会执行 componentWillMount、componentWillUpdate、getDerivedStateFromProps 生命周期函数。因为 render 阶段会被废弃，所以 componentWillMount 等是 unsafe 的。

#### commit 阶段

这一小节，我们主要介绍 commitRoot 的工作内容。

上文已有说明，commit 阶段分为多个子阶段：before mutation 渲染前、mutation 渲染中、layout 渲染后。fiber reconciler 依次会调用 commitBeforeMutationEffects、commitMutationEffects、commitLayoutEffects 函数。对于每一类钩子，fiber reconciler 首先会通过 subtreeFlags 判断是否需要处理子树，若是，递归处理子树中的钩子；然后通过 flags 判断是否需要处理当前节点，若是，执行当前节点的钩子。

在 mutation、layout 子阶段之间，finished-work tree 会转化成 current tree。转化时机点的设置，在于使 componentWillUnmount 处理转化前的 current fiber，componentDidMount、componentDidUpdate 钩子处理转化后的 current fiber。

三阶段处理完成后，会调用 requestPaint 函数，告诉调度器渲染线程需要展开作业。

### 多流程调度

#### scheduler 调度

上文已提到，有两类调度，第一类在 js 执行引擎和渲染线程之间，第二类在 js 脚本任务之间。scheduler 提供了任务调度的底层能力，主要的封装接口包含：

* scheduleCallback(priorityLevel, fn)：按优先级异步调度任务队列。任务会被推入调度队列中，等待渲染线程空闲时展开工作。当任务队列过长或有渲染作业时，并且当前时间小于任务的到期时间点 expirationTime，scheduleCallback 就会把工作交还给渲染线程
* cancelCallback(task)：废弃任务（多任务共享资源，因此当高优先级任务抢占执行时需要废弃低优先级任务）
* runWithPriority(priorityLevel, fn)：更改 currentPriority，同步执行回调。currentPriority 会影响调度任务的优先级
* shouldYield：判断任务执行是否过长或者渲染作业是否在处理中（requestPaint 函数已调用）。scheduleCallback 在切换任务时，会访问 shouldYield，以免渲染线程得不到执行
* requestPaint：通知渲染线程将展开工作，将导致 shouldYield 将返回真值

![image](reconciler6.png)

scheduleCallback 在执行任务队列，即便是 ImmediatePriority 优先级，一个任务执行完后可能会因为渲染线程打断，不能同步执行下一个任务。为此，fiber-reconciler 在 scheduler 之上作了一层封装，它允许使用 scheduleSyncCallback 调度同步任务队列，即该任务队列一旦执行，就会一次性将所有同步任务都执行完（姑且称为“批处理任务”），不会被渲染线程打断。调度指的是，scheduleSyncCallback 会使用 scheduleCallback 调度“批处理任务”。此外，flushSyncCallbackQueue 可以直接执行同步任务队列，它会调用 cancelCallback 废弃 “批处理任务”。同步任务队列执行前，currentPriority 会被 runWithPriority 改写为 ImmediatePriority。

fiber-reconciler  也实现了 scheduler 优先级和 react 优先级的转换等操作。

[React 中的调度](https://github.com/xitu/gold-miner/blob/master/TODO1/scheduling-in-react.md) 中有基于 scheduler 调度低优先级任务的案例 [ScheduleTron3000](https://codesandbox.io/s/71ly7qx83x?from-embed=&file=/src/index.js:1714-1737)，即延迟处理低交互需求的渲染流程。

#### reconciler 调度

上文已提到，将 workLoopSync、workLoopConcurrent 与 commitRoot 串起来就可以发起一次渲染流程，这由 performSyncWorkOnRoot、performConcurrentWorkOnRoot 函数完成。performSyncWorkOnRoot、performConcurrentWorkOnRoot  都可以通过 scheduleCallback 或 scheduleSyncCallback 异步调度。这样一旦渲染线程需要展开工作时，渲染流程就会挂起。等到渲染线程工作完成后，被挂起的渲染流程才可以恢复或废弃（让位给高优先级任务）。直接调用 performSyncWorkOnRoot 能兼容 stack reconciler 的处理方式。

虽然 scheduleCallback 可以调度多任务，fiber-reconciler 内实际只会在根节点上调度一个任务，即 root.callbackNode，此为 ensureRootIsScheduled 函数的表现。performSyncWorkOnRoot、performConcurrentWorkOnRoot 函数也会从根节点发起。这会带来以下几个问题：
* 当使用 setState 重绘某节点时，从根节点发起的渲染流程怎么避过不需要更新的节点？
* 当在 useEffect 内调用多次 setState，怎么保证这几个 setState 都是有效的？

对于第一个问题，使用 setState 重绘时，fiber-reconciler 会为源节点打上 fiber.lanes 标识，表示它有更新作业。在 render 阶段的 beginWork 函数处理期间，它会通过该标识判断节点是否存在更新（即 [includesSomeLane(renderLanes, workInProgress.lanes)](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3115)）。若有，递归处理子树；若无，直接复用原节点，无需递归子树。随后，beginWork 又会将该节点的 lanes 标识重置为 NoLanes（workInProgress.lanes 被重置，currentFiber.lanes 未被重置，因此任务废弃后，重新调度还会执行 fiber 的更新），表示没有更新。这样就可以实现按需更新 fiber 节点树。首次渲染和更新渲染也就可以有同一的调用接口 scheduleUpdateOnFiber。

对于第二个问题，我们通过以下几种模式加以说明：
* legacy 模式：历史的同步模式。setState 重绘时 fiber.lanes 标识将置为 SyncLane（即 [requestUpdateLane(fiber)](https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L403) 表现）。它会将多个 setState 更新收集为一个队列，在一个调度周期中处理更新队列（即 ensureRootIsScheduled 表现的重用 root.callbackNode）
* blocking 模式：setState 重绘时 fiber.lanes 标识将置为 SyncLane 或 SyncBatchedLane（置为何种取决于 currentPriority 是否为 ImmediatePriority。若是，标识为 SyncLane；不是，标识为 SyncBatchedLane）
* concurrent 模式：并发模式。setState 重绘时 fiber.lanes 标识将通过 currentPriority 计算获得，这会导致调度一个低优先级任务

通过将 fiber.lanes 标识为更新，也可以使多个 setState 都表现在 fiber 节点树上。在 concurrent 模式下调度 fiber 任务单元时，如果还没有到 fiber 的到期时间点，更新操作就会放在下一次调度中执行。这样就使 setState 渲染流程有了优先级的保障，低优先级的可以延时处理。为此，为了调度作业的不中断，我们可以看到，调度 root.callbackNode 单个任务的 ensureRootIsScheduled 在 commit 阶段尾部也会被调用。下图是 scheduleUpdateOnFiber 入口的整体流程：

![image](reconciler7.png)

除了 ensureRootIsScheduled 在很多地方可以看见外，flushPassiveEffects 也是经常被调用的，因为 useEffect 可能会添加 ImmediatePriority 优先级的任务，flushPassiveEffects 会调用 flushSyncCallbackQueue 执行同步任务队列。

### 后记

本文仅对 fiber reconciler 付诸逻辑性的解说，对诸多枝节的演绎尤感乏力。作几句话总结：

1. fiber reconciler 以栈帧为原型处理 fiber 节点，它会在 render 阶段完成虚拟节点的更新、钩子的收集，在 commit 阶段以钩子形态完成 dom 树的渲染
2. 为了保证页面刷新的帧率，fiber reconciler 会在指定时间分片让位给渲染线程，这样就允许由 fiber 任务单元构成的 render 阶段可以被打断，以调度高优先级任务

### 参考

[使用 Concurrent 模式（实验性）](https://react.docschina.org/docs/concurrent-mode-adoption.html)
[深入React fiber架构及源码](https://zhuanlan.zhihu.com/p/57346388)
[React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
[A look inside React Fiber](https://makersden.io/blog/look-inside-fiber/)
[React 中的调度](https://github.com/xitu/gold-miner/blob/master/TODO1/scheduling-in-react.md)