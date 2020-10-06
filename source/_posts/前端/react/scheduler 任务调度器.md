---
title: scheduler 任务调度器
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: '54688819'
date: 2020-10-06 06:00:00
updated: 2020-10-06 06:00:00
---

以状态变更驱动的重绘流程为例，状态变更到计算虚拟节点的过程可以被挂起，虚拟节点到渲染真实 dom 的过程却不可以被打断。这就有了调度器的用武空间，如以较低优先级运行有较大运算量的状态更新任务，以较高优先级运行渲染任务；如在多个状态更新任务排队时可以打断和恢复，将工作线程交还给渲染线程等等。[React 中的调度](https://github.com/xitu/gold-miner/blob/master/TODO1/scheduling-in-react.md) 对此介绍了很多。fiber 架构使得 react 中的状态计算任务可以被拆分成很小的单元，以便在渲染线程工作停当的间隙执行状态更新任务，即在多个帧上渲染 react 组件。

如 [React 中的调度](https://github.com/xitu/gold-miner/blob/master/TODO1/scheduling-in-react.md) 已总结的，scheduler 任务调度器首先为任务抽象了以下几种优先级：

* ImmediatePriority：立即执行优先级，需要同步执行的任务
* UserBlockingPriority：用户阻塞型优先级（250ms 后过期），如按钮点击
* NormalPriority：普通优先级（5s 后过期），不必让用户立即感受到的更新
* LowPriority：低优先级（10s 后过期），可以推迟但最终仍然需要完成的任务，如数据分析
* IdlePriority：空闲优先级（永不过期），不必运行的任务，如隐藏界面以外的内容

拥有较高优先级的任务较先执行，反之亦然。250ms 后过期指的是，expirationTime 为当前时间加 250 ms。expirationTime 影响任务的执行时间点，其值小的任务会排在任务队列的前面，较先执行；其值大的任务会排在任务队列的后面，较晚执行。计算 expirationTime 时加上当前时间，是为了避免饥饿——低优先级的任务永远得不到执行。
其次，scheduler 需要对接浏览器、native 环境等。在这方面，它制作了不同的 SchedulerHostConfig 模块，以便对接浏览器环境中特有的 requestAnimationFrame、MessageChannel 等，或使用 setTimeout 等。在浏览器环境中，scheduler 实际使用 MessageChannel 制作 SchedulerHostConfig 模块，因为 MessageChannel 在 web worker 中也可以使用。scheduler 使用宿主环境提供的接口执行任务。此外，任务长期工作会导致渲染线程迟迟没法工作，因此 scheduler 需要适时让出工作线程（这里的任务包含运行时间长的任务，也包含运行时间短的多个任务）。

除了在调度器中协调任务的优先级（主要是 unstable_scheduleCallback 函数），允许任务被挂起外，有些任务是不可被打断的。因此调度器内还提供了与任务调度无关的回调执行流程，如 unstable_next、unstable_runWithPriority。

本文分为三个小节介绍 scheduler，分别说明以下三个问题：

1. scheduler 怎样使用 SchedulerHostConfig 模块对接宿主环境？
2. scheduler 怎样使用 taskQueue、timerQueue 抽象任务及任务队列？
3. scheduler 怎样协调任务的执行？

### 对接宿主环境

SchedulerHostConfig 模块包含一些预留方法，分别由不同环境提供具体实现。我们先来看一下 [SchedulerHostConfig.default.js](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/forks/SchedulerHostConfig.default.js) 模块是怎样实现这些预留方法的。首先，这个模块通过判断 window 或 MessageChannel 全局变量判断环境，MessageChannel 也是浏览器端接口。（注：[SchedulerHostConfig.default.js](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/forks/SchedulerHostConfig.default.js) 模块目前没有使用 requestAnimationFrame，但是会校验浏览器是否支持 requestAnimationFrame，未来不排除使用）

* getCurrentTime：获取脚本运作时间（脚本执行时间-程序启动时间）。实现上基于 performance 或 Date
* requestHostCallback：native 环境通过 setTimeout 执行回调。若多次调用，会使用 setTimeout 将后续的回调添加到队列中，等待执行。浏览器环境通过 MessageChannel 消息通信触执行回调，若多次调用，后续的回调会被忽略（[SchedulerHostConfig.default.js](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/forks/SchedulerHostConfig.default.js) 允许通过回调返回 true，告诉 MessageChannel 还有未执行的任务，MessageChannel 就会再次发消息）。无论浏览器环境，还是 native 环境，回调的参数 hasRemainingTime 都是 true，currentTime 都是回调执行时间
* cancelHostCallback：将回调置为 null，但不会影响排队的回调
* requestHostTimeout：通过 setTimeout 延时执行回调
* cancelHostTimeout：通过 clearTimeout 取消 requestHostTimeout 设置的定时器
* shouldYieldToHost：native 环境返回 false。浏览器环境限制回调的可执行时长 yieldInterval（默认为 5ms），若有调用过 requestPaint 或超出阈值 maxYieldInterval（默认为 125ms），则让出（yield）工作线程（让位给渲染线程）
* requestPaint：当 requestPaint 被调用后，一旦回调超出可执行时长，等到回调执行完成后，就会让出工作线程。否则要到超出阈值时间时，才会让出工作线程
* forceFrameRate(fps)：native 环境为空函数；浏览器环境根据帧率设置回调的可执行时长

可以看到，SchedulerHostConfig 模块限制了回调（即任务）的可运行时长 deadline = currentTime + yieldInterval，过长时让出工作线程，这样就能保证我们实现动效时的不卡顿。除此而外，SchedulerHostConfig 模块提供了执行同步或异步任务、取消执行这些任务的接口。

### 任务队列

任务队列表现为 Heap 堆（参见 [SchedulerMinHeap.js](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/SchedulerMinHeap.js)），任务在 Heap 堆以 sortIndex、id 属性从小到大进行排序。

* peek：取出首个任务，且该任务不会从队列中移除
* pop：取出并移除首个任务，且会导致任务队列的重排
* push：将任务添加队列中，并导致任务队列的重排

重排采用二分法。因为执行任务时只需保证头部任务的优先级，所以 push、pop 方法的重排逻辑实际上并不能保证队列中任务严格按照 sortIndex、id 从小到大进行排序，而只会保证头部任务的排序是合理的。

在 Heap 堆之上，taskQueue 队列用于存放将被处理的任务；timerQueue 队列用于存放延期处理的任务。[advanceTimers(currentTime)](https://github.com/facebook/react/blob/master/packages/scheduler/src/Scheduler.js#L81) 函数会借助 peek、pop，用于从 timerQueue 队列中取出 startTime 已符合执行条件的任务（即 startTime 大于等于 currentTime），并将该任务移入 taskQueue 队列中。对于 taskQueue 队列中的任务，首先通过 peek 函数取出执行，执行完成后通过 pop 函数移除该任务。

除了 sortIndex、id 之外，taskQueue、timerQueue 队列中的任务包含以下属性：

* callback：任务执行函数
* priorityLevel：任务的优先级，从高到低分别为 ImmediatePriority、UserBlockingPriority、NormalPriority、LowPriority、IdlePriority。优先级影响任务的 expirationTime；优先级越高，expirationTime 越小
* startTime：任务的假定执行时间，默认为当前时间。实际是添加到 taskQueue 队列中的事件。通过 delay 选项设置延时，该任务会先添加到 timerQueue 队列中，delay 时间一过，再移入 taskQueue 队列中
* expirationTime：到期时间，即 options.timeout 或  timeout + startTime（timeout 通过 priorityLevel 计算）。若未设置 options.timeout、priorityLevel，timeout 默认使用 NORMAL_PRIORITY_TIMEOUT = 5000，即 taskQueue、timerQueue 中的任务都有 expirationTime

对于 taskQueue 队列中的任务，sortIndex 的值为 expirationTime，即根据 expirationTime 协调任务的执行顺序，这也是 expirationTime 与优先级 priorityLevel 密切相关的原因。对于 timerQueue 队列中的任务，sortIndex 的值为 startTime，即根据 startTime 协调任务挪移到 taskQueue 的顺序。当任务从 timerQueue 队列挪动到 taskQueue 队列时，该任务的 sortIndex 仍旧会被设置为 expirationTime，即以 expirationTime 协调任务的执行顺序。因此，taskQueue 可视为即将执行的任务队列；timerQueue 可视为排队等待执行的任务队列。同时，timerQueue 中的任务会延时添加到 taskQueue 并执行，因此也可视为延时任务队列；taskQueue 可视为即时任务队列。

expirationTime 设置为 options.timeout 或由 priorityLevel 计算获得。即如 timeout 选项的语意，它指的是任务的预期执行时长。就像最短作业优先调度算法，对于同一时间待调度的任务，较低 expirationTime 的任务在 taskQueue 位置较前，最先得到调度。为了避免低优先级的任务得不到执行（即饥饿），计算 expirationTime 时会结合 currentTime，即后添加的高优先级任务未必会比先添加的低优先级任务拥有更小的 expirationTime。
结合 SchedulerHostConfig 模块限制的任务可执行时长 deadline，若 expirationTime 小于 deadline，任务就会有足够的时间执行；若大于，渲染线程就会长期得不到工作，因此在任务执行完成后，就会将工作线程交还给渲染线程。优先级越高，expirationTime 越小，连续不跌的多个任务反会长期占用工作线程，任务的内容与渲染线程也越紧密；反之亦然。如 ui 渲染型任务需要快速地反应成视图变更，expirationTime 相应也小；状态计算型任务的 expirationTime 相应也大。多个 ui 渲染型任务也需要连贯一致地执行完成，不能被挂起，否则渲染内容就会与预期不一致；多个状态计算型任务却可以被挂起，由内存记录最新的状态，等待渲染线程工作完成后，再接着这个状态执行计算任务。

### 任务执行

看完 scheduler 对任务的抽象，我们会问，任务分别是什么时候添加到 taskQueue、timerQueue 队列中的呢？任务又是怎么执行的呢？

[unstable_scheduleCallback(priorityLevel, callback, options)](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L295) 函数给了我们答案。首先通过选项 options.delay 可以区分即时任务和延时任务，这样就可以将即时任务添加到 taskQueue 中，将延时任务添加到 timerQueue 中。简单地看，即时任务可使用对接宿主环境的 requestHostCallback 函数执行，延时任务可使用 requestHostTimeout 函数执行。

问题是，当一个即时任务在执行期间，我们又通过 unstable_scheduleCallback 创建了另一个即时任务，这时需要怎么处理呢？上文已有描述，浏览器环境默认的 requestHostCallback 是抢占式的，即后续的即时任务会踢掉先前的即时任务，这不符合我们的需求。我们的需求是非抢占式的，后续的任务需等待先前的任务执行完成，即一个即时任务执行完成后，再从 taskQueue 队列中获取下一个任务。在这方面，scheduler 使用 [workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 函数循环处理 taskQueue 队列。我们可以看到，在循环处理流程中，如果任务长期占用工作线程，以致于渲染线程得不到工作（即 currentTime 已经过了 deadline 时间点，还没过 expirationTime 时间点），后续的任务就会被挂起，交回渲染线程进行处理，同时告诉 MessageChannel 还有未执行的任务。

为了避免饥饿，在 [workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 循环处理 taskQueue 队列前或中，它会调用 advanceTimers 将 timerQueue 中的任务移入 taskQueue 中。上文已经提到，startTime 时间一过，timerQueue 待执行任务队列中的任务就可以移入 taskQueue 将执行任务队列中，两者统一经由 taskQueue 执行。在 [workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 循环处理 taskQueue 队列后，如果 taskQueue 队列空了，[workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 就会调用 requestHostTimeout 函数执行 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105)。等 startTime 时间一过，timerQueue 中的任务就会被移入 taskQueue 中，继而再使用 [workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 循环处理。如果 timerQueue 到期需执行的任务不幸是无效的，[handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 就会递归调用自身，以处理 timerQueue 中下一个任务。因此，timerQueue 中的延时任务可以统一由 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 进行处理。

简单地总结上述两段：[handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 用于处理 timerQueue 中的延时任务；[workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 会循环执行 taskQueue 中的即时任务，等到 taskQueue 处理完成后，再调用 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理延时任务。假设 unstable_scheduleCallback 创建的任务都是异步的，它会先等待 startTime 时间点，在这个过程中，如果我们再使用 unstable_scheduleCallback 创建即时任务，scheduler 就会撤销 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理，先处理即时任务，这在 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 函数中有所体现。[workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 也会通过 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 调用，并处理调用失败的任务。我们可以认为，[flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 用于处理 taskQueue 中的即时任务。对于一到 startTime 时间点的延时任务，[handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理时也会调用 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122)。这就是同、延时任务处理上的首尾相衔。

因此，这里总结下 unstable_scheduleCallback 的整体处理流程（下述，使用 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 处理指的是通过 requestHostCallback 调用 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122)；使用 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理指的是通过 requestHostTimeout 调用 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105)；撤销 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理指的是调用 cancelHostTimeout）：

* 首次添加即时任务，使用 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 处理，设置 isHostCallbackScheduled 标识
* 首次添加延时任务，使用 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理，设置 isHostTimeoutScheduled 标识
* 即时任务处理中，再次添加即时任务，再次使用 [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 处理
* 即时任务处理中，再次添加延时任务，只将任务添加到 timerQueue 队列中
* 延时任务处理中，再次添加即时任务，撤销 [handleTimeout](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L105) 处理，转而使用  [flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122) 处理
* 延时任务处理中，再次添加延时任务，只将任务添加到 timerQueue 队列中

以下是 unstable_scheduleCallback 函数的简要处理流程：

![image](scheduler.png)

除了 unstable_scheduleCallback 函数外，scheduler 还提供了以下接口：

* unstable_getFirstCallbackNode：获取 taskQueue 队列中的首个任务
* unstable_getCurrentPriorityLevel：获取 currentPriorityLevel 缓存，其值为 [workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 循环中当前任务的执行优先级
* unstable_pauseExecution：暂停任务。通过在 [workLoop](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L164) 循环中织入 isSchedulerPaused = true 标识实现
* unstable_continueExecution：恢复任务，将 isSchedulerPaused 置为 false，并尝试重新执行 requestHostCallback([flushWork](https://github.com/facebook/react/blob/v16.13.0/packages/scheduler/src/Scheduler.js#L122))
* unstable_cancelCallback(task)：终止任务，将 task.callback 置为 null
* unstable_shouldYield：调用 SchedulerHostConfig 模块的 shouldYieldToHost 接口，任务是否运行过长
* unstable_requestPaint：调用 SchedulerHostConfig 模块的 requestPaint，超出可执行时长的任务完成后，工作线程将让位给渲染线程
* unstable_next(eventHandler)：以 NormalPriority 或更低优先级执行 eventHandler，比如用于执行大计算量的分析型任务
* unstable_runWithPriority(priorityLevel, eventHandler)：以指定优先级执行 eventHandler

[React 中的调度](https://github.com/xitu/gold-miner/blob/master/TODO1/scheduling-in-react.md) 这篇文章中有 unstable_next、unstable_runWithPriority 的使用示例。

综上，scheduler 提供的最主要接口为：

* unstable_scheduleCallback：以优先级调度的方式执行任务
* unstable_runWithPriority：以指定优先级执行回调

### 其他

#### 内部使用

react-reconciler 使用 SchedulerWithReactIntegration.js 对接 scheduler 模块，然后被显示调用。简要地讲，首先 react 内部优先级需要转化成 scheduler 优先级，其次 SchedulerWithReactIntegration.js 支持同步任务队列 syncQueue（优先级为 ImmediatePriority），它会调用 unstable_runWithPriority 一次性执行同步任务，并提供 scheduleSyncCallback 接口用于调度同步任务，最后即是对 scheduler 模块其他接口的包装。

#### 不足

React 中的调度 提到了 scheduler 模块的不足：不同的调度器实例在使用资源时会出现竞争关系；浏览器机制中的渲染、垃圾回收会使调度器无法正常工作。这方面，chrome 正在提供新的接口。我们也可以看到，SchedulerPostTask.js 模块正是基于 chrome 的新接口，完成调度作业的。

### 后记

本文仍显得过多着眼细节，未如 [React 中的调度](https://github.com/xitu/gold-miner/blob/master/TODO1/scheduling-in-react.md) 聚焦而透彻。