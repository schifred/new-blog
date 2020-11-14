---
title: hooks 演绎录：useMutableSource
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: '11393516'
date: 2020-11-14 06:00:00
updated: 2020-11-14 06:00:00
---

useMutableSource 用于监听外部数据源的变动，在数据源变动时，它会调度重绘任务。useMutableSource 包含三个参数：

```js
useMutableSource(source, getSnapshot, subscribe);
```

* source 声明外部数据源，通常通过 createMutableSource 函数创建，除了 source._source 指实际数据源之外，还包含 getVersion 获取数据源相关的版本号
* getSnapshot 声明对数据源的哪部分数据感兴趣
* subscribe 在数据源变动时订阅指定数据

首先看一下 [useMutableSource 使用文档](https://github.com/bvaughn/rfcs/blob/useMutableSource/text/0000-use-mutable-source.md) 中结合 redux 的使用示例：

```js
// Somewhere, the Redux store needs to be wrapped in a mutable source object...
const mutableSource = createMutableSource(
  reduxStore,
  // Because the state is immutable, it can be used as the "version".
  () => reduxStore.getState()
);
// It would probably be shared via the Context API...
const MutableSourceContext = createContext(mutableSource);
// Because this method doesn't require access to props,
// it can be declared in module scope to be shared between hooks.
const subscribe = (store, callback) => store.subscribe(callback);
// Oversimplified example of how Redux could use the mutable source hook:
function useSelector(selector) {
  const mutableSource = useContext(MutableSourceContext);
  const getSnapshot = useCallback(store => selector(store.getState()), [
    selector
  ]);
  return useMutableSource(mutableSource, getSnapshot, subscribe);
}
```

在有 useState、useEffect 的基础上，我们能否实现上述功能呢？答案是可以，简要实现代码如下：

```js
const useReduxStore = (selector) => {
    const [snapshot, setSnapshot] = useState(selector(reduxStore.getState()));
  
  useEffect(() => {
    return reduxStore.subscribe(() => {
      const newSnapshot = selector(reduxStore.getState());
      if (newSnapshot != snapshot){
        setSnapshot(newSnapshot);
      };
    });
  }, [reduxStore, selector]);
  
  return {
    snapshot,
  };
}
```

那么，fiber reconciler 中的 useMutableSource 是怎么实现的呢？useMutableSource 实现上也会包含上述逻辑，即在 useEffect 过程中绑定 subscribe 监听函数，当 source 数据 getSnapshot 部分变更时，调用 setState 更新 fiber reconciler 内部管理的 currentSnapshot 状态。除此之外，useMutableSource 还包含以下两个逻辑：

* 当 getVersion 获取的版本号变更时，通过 useEffect、setState 机制更新 currentSnapshot 状态
* 当 source、getSnapshot 或 subscribe 变更时，重新计算 snapshot。与此同时，对于原 source 的部分数据更新，程序上需要丢弃对该更新内容的响应，因此 useMutableSource 会清除内部 hook.queue 队列（该 hook 由创建 currentSnapshot 状态的 useState 生成）