---
title: hooks 演绎录：useEffect、useLayoutEffect
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: c4c220d7
date: 2020-10-31 06:00:00
updated: 2020-10-31 06:00:00
---

依循交互需求，视图框架需要在渲染前、后执行指定的钩子。useEffect、useLayoutEffect 钩子的表现即如此，一个会在渲染完成后异步执行，另一个同步执行。它们是我们惯常所理解的钩子——在处理流程的指定时机执行。useState、useReducer 在表现上是变更状态，为什么它们也被冠名为钩子呢？react hooks 会穿插在 fiber reconciler 协调渲染流程的过程中执行，即它们在内部实现中有钩子的特征。
本文将介绍 fiber reconciler 机制下 useEffect、useLayoutEffect 的实现。阅读本文，您需要先了解 [fiber reconciler 漫谈](/archives/7c21db8b/)、[hooks 演绎录：useState、useReducer](/archives/4fcadd8a/)。

### 简要

一般钩子系统的处理流程大体分为三步：声明钩子、收集钩子、执行钩子。假如我们要为函数 fn 实现一套简单的钩子机制，在 fn 执行前调用 before 钩子，执行后调用 after 钩子。我们可以在映射表中维护 fn 与钩子函数的关联关系，然后通过代理就能在这个函数执行前、后调用钩子函数。在对接层面，既可以暴露接口用于注册 fn 的钩子，又可以使用装饰器声明钩子。

我们把思绪转到 react 类组件，它可以有 componentDidMount、componentDidUpdate 等生命周期钩子。其实现，无非是先声明组件实例包含哪些生命周期钩子，然后将组件实例转化为内部表示（fiber 节点），在协调渲染流程的过程中取出钩子并执行。如果我们对 fiber reconciler 的实现有所了解，就会知道 fiber.stateNode 即类组件实例，因此在协调渲染流程的过程中，程序就可以显式调用 stateNode.componentDidMount 了。

回到 react hooks 机制，使用 useEffect、useLayoutEffect 可以为函数式组件制作钩子函数。useEffect、useLayoutEffect 的钩子函数都会以 effect 对象的形式存入 fiber.updateQueue 链表中。在 fiber reconciler 协调渲染流程的过程中，effect 对象会被取出并执行。useLayoutEffect 与实际的渲染动作挂钩，即渲染前后调用。useEffect 与 fiber reconciler 中的优先级调度算法有关，它可以指定不同的优先级，既可能在本次调度周期中即时执行，又可能在下一次调度周期中延时执行。

![image](useEffect1.png)

### 详解

effect 对象的属性包含 tag 类型、deps 依赖对象、create 首渲或依赖对象变更时执行函数、destroy 卸载时执行函数（作为 create 函数的返回值）。通过 useEffect、useLayoutEffect 制作的钩子函数都可以转化成 effect 对象，两者有不同的 tag 值。因为都可以转化成 effect 对象，useEffect、useLayoutEffect 就可以共用同一套添加钩子、执行钩子的逻辑：

* 通过 pushEffect 函数将 effect 对象添加到 fiber.updateQueue 链表中
* 通过 commitHookEffectListMount 执行 fiber.updateQueue 链表中指定 tag 属性的 effect.create，并获得 effect.destroy
* 通过 commitUnmount 或 commitHookEffectListUnmount 执行 fiber.updateQueue 链表中指定 tag 属性的 effect.destroy

#### render 阶段

render 阶段，useEffect、useLayoutEffect 会在 renderWithHooks 包装函数内通过 ReactCurrentDispatcher 切换具体功能。其代码结构如下图，与 useState、useReducer 的相类。mountEffectImpl、updateEffectImpl 都通过 pushEffect 将 effect 对象推入 fiber.updateQueue 链表中。

![image](useEffect2.png)

通过 fiber.flags、fiber.subtreeFlags 能快速判断组件或子树是否包含钩子以及某类钩子，因此不必遍历 fiber.updateQueue 链表。对于使用 useEffect 添加的 effect 对象，其 flag 值会包含 HookPassive，下文简称 HookPassive 类钩子；对于使用 useLayoutEffect 添加的 effect 对象，其 flag 值会包含 HookLayout，下文简称 HookLayout 类钩子。当 flags 包含 HookHasEffect 时，意味着有钩子需要被执行。不然，即便 flags 包含  HookPassive 或 HookLayout，钩子也不会被执行，这样处理是为了应对 deps 依赖对象未变更的场景。

effect 对象也会存入 fiber.memoizedState 链表中。只有这样，程序才能在 renderWithHooks 包装函数执行期间，取出之前的 effect 对象（通过双缓冲技术从 current fiber 中获取），再比较 deps 依赖对象是否有变更。如果 deps 依赖对象没有变更，就重用之前的 effect 对象内容；如果变更了，就创建一个新的 effect 对象，并使 effect.tag 包含 HookHasEffect。

#### commit 阶段

先来聊聊 HookLayout 类钩子，因为它和渲染操作强关联，会在 commit 阶段一次性执行完成。对应于 commit 的三个子阶段：beforeMutation 渲染前、mutation 渲染中、layout 渲染后。effect.create 会在 layout 子阶段执行，并创建 effect.destory；effect.destroy 会在 mutation 子阶段执行，因为组件的卸载也会在这一阶段完成。实现上，
* layout 子阶段的处理函数—— commitLayoutEffects 会间接调用 commitHookEffectListMount，以取出 HookLayout 类钩子且执行 effect.create
* mutation 子阶段的处理函数——commitMutationEffects 会间接调用 commitUnmount，以取出 HookLayout 类钩子且执行 effect.destory，随后卸载组件。这一阶段的卸载流程依赖于 render 阶段 reconcileChildren 协调子节点的过程中收集 fiber.deletions——移除的子节点

当然，无论在 mutation 还是 layout 子阶段，fiber reconciler 会先行递归处理子树中的节点。

![image](useEffect3.png)

再来聊聊 HookPassive 类钩子，它与优先级调度算法有关。从源代码中能发现：

* flushPassiveUnmountEffects 函数会间接调用 commitHookEffectListUnmount 函数，以取出 HookPassive 类钩子且执行 effect.destroy
* flushPassiveMountEffects 函数在执行过程中会对子树中被删除的节点递归使用 commitHookEffectListMount 函数，以取出 HookPassive 类钩子且执行 effect.create

当然，flushPassiveUnmountEffects、flushPassiveMountEffects 均会先行递归处理子树中的节点。flushPassiveUnmountEffects、flushPassiveMountEffects 统一通过 flushPassiveEffects 函数调用。 在下一小节中，我们会详细介绍 flushPassiveEffects。

#### flushPassiveEffects

useEffect 内可以使用两类语句：setState 等状态变更类语句，非状态变更类语句。setState 等状态变更类语句会引起异步调度；非状态变更类语句会同步执行。对于 setState 的调用，useEffect 也可以采用两种方式：常规调用方式——useEffect 内直接调用 setState 更新状态；指定优先级调用方式——useEffect 内使用 runWithPriority 指定 setState 更新状态的优先级（指定的优先级可以是 ImmediatePriority 优先级）。

为了说明常规调用方式、指定优先级调用方式的差别，我们先需要知道 fiber reconciler 渲染包含 legacy 同步模式、concurrent 并发模式两种。同步模式下，所有状态更新都会置为 ImmediatePriority 优先级；并发模式下，状态更新任务可以置为不同的优先级。不同的调用方式会与不同的渲染模式结合。

考虑一下 useEffect 创建的 effect.create、effect.destroy 钩子函数，它们通常可以在渲染后异步调度执行。我们可以在 before mutation 子阶段以及 layout 子阶段后，看见 fiber reconclier 会执行以下代码：

```js
if (!rootDoesHavePassiveEffects) {
  rootDoesHavePassiveEffects = true;
  scheduleCallback(NormalSchedulerPriority, () => {
    flushPassiveEffects();
    return null;
  });
}
```

那么，为什么需要在 before mutation 子阶段以及 layout 子阶段后都调度 flushPassiveEffects？before mutation 子阶段中，fiber reconciler 会通过子树中节点的 falgs 属性，判断该节点是否包含 HookPassive 类钩子。layout 子阶段后，fiber reconciler 会通过根节点的 falgs 以及 subtreeFalgs 属性，判断跟节点是否包含 HookPassive 类钩子，且首层子节点是否被删除。rootDoesHavePassiveEffects 作为 flushPassiveEffects 在调度中的标识，它用于保证 flushPassiveEffects 在 before mutation 子阶段或 layout 子阶段后仅调度一次。

我们看到，flushPassiveEffects 会以 NormalPriority 优先级进行调度。但是，这样也会把 currentPriority 置为 NormalPriority，无法满足以下两个场景：
* 并发模式需要以不同优先级执行 useEffect 内的状态更新任务
* 历史的同步模式需要立即执行 useEffect 内的状态更新任务

我们看到，flushPassiveEffects 函数内部作出了处理：useEffect 内状态更新任务的优先级取决于重绘任务的优先级，当其大于 NormalPriority，则取 NormalPriority；否则取重绘的优先级。因此，对于同步模式，状态更新任务使用 ImmediatePriority 优先级；对于异步模式，状态更新任务取重绘的优先级，这通常是 NormalPriority 优先级。此外，
* 同步模式，flushPassiveEffects 函数尾部会调用 flushSyncCallbackQueue 执行同步任务队列，即立即处理同步模式产生的状态更新。如果 useEffect 内的状态更新任务引起了另一个 useEffect 作业，
* 异步模式，它会有内外两层优先级，首先以 NormalPriority 优先级调度 flushPassiveEffects，其次以不同的优先级调度 setState 更新

假想在同步模式下，useEffect 内的状态更新任务导致了另一个 useEffect 作业的执行，后者也产生了一个 ImmediatePriority 优先级的状态更新任务，这时会怎么处理呢？对此，fiber reconciler 在 commit 阶段的起始会通过 while 循环调用 flushPassiveEffects 以执行同步任务队列，将多个 useEffect 的状态更新归并为一次视图重绘。并发模式不必过分考虑 useEffect 间的相互影响，它可以等待一次视图重绘完成之后再执行下一个状态更新任务。

我们看到，useEffect 内状态更新任务在并发模式下都是异步调度；在同步模式下首次为异步调度，其余互为影响的 useEffect 再次为同步执行。假想 useEffect 内的状态更新任务在异步调度过程中，即等待执行阶段，这时程序再触发一个高优先级的状态更新任务，我们需要保证 useEffect 内的状态更新任务在优先级足够时（到期或 ImmediatePriority 优先级 ）优先执行。为此，fiber reconciler 会在 performConcurrentWorkOnRoot、performSyncWorkOnRoot 启动 render 阶段处理前，调用 flushPassiveEffects 执行 useEffect 内的状态更新任务。

综上，flushPassiveEffects 的执行时机包含：
* performConcurrentWorkOnRoot、performSyncWorkOnRoot 执行头部，同步执行 flushPassiveEffects，以免另一个渲染流程抢占 useEffect 钩子
* commit 阶段头部，循环同步执行 flushPassiveEffects，一次性同步处理多个 ImmediatePriority 优先级的 useEffect 钩子
* beforeMutation 子阶段执行时（当子树中节点有 HookPassive 类钩子），以 NormalPriority 优先级调度 flushPassiveEffects
* layout 子阶段执行后（当子树或根节点有 HookPassive 类钩子且有子节点卸载），以 NormalPriority 优先级调度 flushPassiveEffects

![image](useEffect4.png)
