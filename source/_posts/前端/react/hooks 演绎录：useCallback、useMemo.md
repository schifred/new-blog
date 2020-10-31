---
title: hooks 演绎录：useCallback、useMemo
category:
  - 前端
  - react
tags:
  - 前端
  - react
keywords: react
abbrlink: 43315c36
date: 2020-10-31 12:00:00
updated: 2020-10-31 12:00:00
---

一般，我们会在函数式组件中创建绑定函数，如：

```js
export default function Demo() {
  // 此处省略八百字...
  const onClick = () => {
    console.log('谁动了我的奶酪');
  };
  return (
    <>
      {/* 此处省略八百字... */}
      
        <button onClick={onClick}>点我</button>
    </>
  );
}
```

App 组件一旦重绘，就会重新创建 onClick 函数，这至少会引起 gc 消耗、props 比对消耗。笔者不确信这样处理会不会引起 button 节点的重绘消耗。假想一种事件处理机制：通过委托可以在根节点绑定事件，再由 event.target 找到对应的 fiber，随后调用 fiber 节点的绑定函数。这时，onClick 并不需要作为注入 dom 节点的 attributes。

使用 useCallback 可以解决上述问题，即它能缓存函数。当依赖对象没有变化时，就从缓存中取出原函数，若变化，则将匿名函数及依赖对象作为 memoizedState 属性（[callback, deps] 的形式）存入 fiber.memoizedState 队列。从 fiber reconciler 的实现看来，useCallback 也更像一个语法糖，跟协调渲染的主流程无甚关系。

除了缓存函数之外，我们也可以使用 useMemo 缓存数据（特别是根据某个状态值变更的计算属性，只在相关状态变更时重新计算）。useMemo 的实现与 useCallback 相类。以下贴示两者的简要示例：

```js
export default function Demo() {
  const [name, setName] = useState(undefined);
  const [sex, setSex] = useState(undefined);
  // 使用 useCallback 避免 sex 更新引起重新创建函数
  const callback = useCallback(() => {
    console.log(`name: ${name}`);
  }, [name]);
  // 使用 useCallback 避免 sex 更新引起的重新计算
  const capitalizedName = useMemo(() => {
    return name.charAt(0).toUpperCase() + name.slice(1);
  }, [name]);
  
  return (
    <>
      {/* 此处省略八百字... */}
        <>
            <label>name:</label>
            <input onChange={(e) => {setName(e.target.value)}} />
        </>
  
        <>
            <label>sex:</label>
            <input onChange={(e) => {setSex(e.target.value)}} />
        </>
        <>
            <label>capitalizeName:</label>
                <span>{capitalizedName}</span>
            </>
      
        <button onClick={onClick}>点我</button>
    </>
  );
}
```