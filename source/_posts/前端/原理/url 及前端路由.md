---
title: url 及前端路由
category:
  - 前端
  - 浏览器机制
tags:
  - 前端
  - 浏览器机制
keywords: 浏览器机制
abbrlink: 4e324a49
date: 2020-12-22 06:00:00
updated: 2020-12-22 06:00:00
---

### 作为资源位的 url

试想一个简单的渲染器，它需要根据标识，切换显示的内容。url 就等同于这种标识，它的学名也叫“统一资源定位器”。在没有 ajax 的年代，我们每次键入不同的 url，浏览器就会获取不同的远程资源，完成 html 渲染、js 执行等等。浏览器也提供了回退、前进、刷新按钮，这几种按钮可以免于键入 url。在 js 脚本中，我们也可以通过 location.href 或 window.open 整页刷新以获取新的资源。

![image](url1.png)

有了 ajax 之后，我们告别了整页刷新——即时获取远程资源（或数据），页面只局部刷新。与 ajax 相同，当使用 history.pushState、history.replaceState 切换 url 时，浏览器也不会刷新整页，我们可以根据更新后的 url 请求远程数据，刷新页面的局部内容。这为我们实现单页应用提供了便利。

参考 history 文档，我们知道，history 对象内维护了地址栏的页面路由栈。我们可以通过 history.forward()、history.back()、history.go(num) 进行跳转到前一页、后一页等。实际上，浏览器的回退、前进按钮也基于这几个接口。

本文将 url 变更分为两类：

* 栈更新类：即使用 history.pushState、history.replaceState 往页面路由栈中插入或替换栈顶数据。没有可泛用的监听事件。但如果，history.pushState 针对 hash 路由模式 url 时，可以监听 hashchange 事件
* 栈重用类：即使用 history.forward、history.back、history.go 或点击回退、前进按钮，重用页面路由栈中数据，可监听 popstate 事件。当页面路由栈条目数变更时，就会触发 popstate 事件。监听 popstate 可以改写回退键的行为

![image](url2.png)

history.pushState(state, title, url) 、history.replaceState(state, title, url) 中的参数 state 是附带的状态值。通过 history.state 可以访问栈顶的状态值，它的最大存储容量为 640kb。

#### url 构成及前端路由模式

```js
protocol//host[:port][/path][?search][#hash]
```

一个典型的 url 包含 protocol 协议，host 域名，port 端口，path 页面路径，search 查询字符串，hash 路径。

前端单页应用的常见路由模式包含两种：

* browser 路由：指前端单页应用通过变更 pathname 切换视图内容，没有可监听事件
* hash 路由：指前端单页应用通过变更 hash 切换视图内容，变更时会引起 hashchange 事件

无论以上两种路由模式的哪一种，前端单页应用都需要在 url 变更后，动态切换视图展示内容。出于体验的需要，我们仅能通过 history.pushState、history.replaceState 切换 url，而不能选择 location.href、window.open，那会造成整页刷新。不巧的是，history.pushState、history.replaceState 并没有匹配的可监听事件。因此，为了确切地感知 url 变更的时机，前端路由框架通常会封装 history.pushState、history.replaceState，在这两个接口调用后，再执行绑定在路由框架上的 listener。这些 listener 的典型内容就包含：通知视图层根据最新的 url 更新页面内容。

![image](url3.png)

对于这一套围绕 url 变更的事件模型，本文稍后会介绍，react 系列会在 history 库中实现，然后在 react-router 中直接使用这套事件模型，在 url 变更后驱动视图内容更新。直观地看，history、react-router 体现了分层，vue-router 却选择将这些糅在一个模块中处理。本文仅介绍 history、react-router。

### history

[history](https://github.com/ReactTraining/history/tree/v5.0.0) 提供了三种路由模式：

* browser 路由：通过 createBrowserHistory 创建 history 对象，该对象会基于 history.pushState 变更 path 页面路径
* hash 路由：通过 createHashHistory 创建 history 对象，该对象会基于 history.pushState 变更 hash 路径
* memory 路由：通过 createMemoryHistory 创建 history 对象，该对象只会变更缓存

三种 history 对象包含相同的属性与方法。与 window.history 相同，history 对象也维护着虚拟的页面路由栈。每个栈中的内容表现为 Location = { pathname, search, hash, key, state } 地址对象形式。history.push 将访问记录压入栈顶；history.replace 替换栈顶内容；history.go 根据栈中数据前进或回退。

上文已有说明，history 内部实现了一套简易的事件模型——通过 history.listen 绑定监听函数 listener；history.push、history.replace 方法执行期间会调用这些监听函数。监听函数的形式为 ({ action, location }) => void。action 的值包含 Push 入栈、Replace 替换、Pop 活动历史记录的总条数变更。

如同 action 的值包含 Pop，当用户点击浏览器中的前进、回退按钮时，理论上也需要调用监听函数，这样才会促使页面更新。所以，history 在处理 browser、hash 路由时会监听 popstate 事件，触发监听函数 listener 执行；memory 路由同样会触发 listener 执行。

![image](url4.png)

除了监听函数 listener 外，history 还允许设置 blocker 拦截器，阻断 url 的变更。官方给出的典型场景如当表单数据变更后，点击回退按钮，这时就可以通过 blocker 校验表单数据是否更新，如更新，阻断 url 变更，提醒用户保存；如未更新，url 正常变更。

history 模块还提供了 createPath、parsePash 接口，用于创建和解析路径。

### react-router

[react-router](https://github.com/ReactTraining/react-router/tree/v5.2.0) 首先会基于 history 库创建指定的 history 对象，这样就可以在 history 对象上绑定监听函数，以便驱动视图重绘。react-router 中的 Router 组件即会将 history 对象作为 props 属性，以此绑定监听函数，将最新的 Location 对象通过 context 机制传入下级组件。这些下级组件包含 Switch、Route、Redirect 组件。

![image](url5.png)

Route 组件与特定的 path 路径绑定，指浏览器地址为该路径，渲染 Route 组件内容。Route、Redirect 组件会作为 Switch 的子组件。因此，当 Switch 组件取得 location 对象后，就可以根据 location 切换渲染指定的 Route 内容了。Redirect 组件也是一样的，当它的 props.path 属性匹配 location 对象时，它就会执行 history.push 或 history.replace 方法跳转。
在 Router 组件的基础上，BrowserRouter、HashRouter、MemoryRouter 等会与特定 history 挂钩的 Router 组件。Link 组件会渲染供跳转使用的 a 链接。

此外，react-router 还提供了：

* useHistory、useLocation、useParams、useRouteMatch：hooks，用于获取 context 传递的数据
* withRouter(Component)：包装 Component 组件，使 Component 组件可以通过 props 访问 location 信息
* matchPath(pathname, { path, exact?, strict?, sensitive? })：路径 pathname 匹配路由模式 path 的情况，返回 match = { path, url, isExact, params } 对象。实现基于 path-to-regexp

### react-router-cache-route

react-router-cache-route 提供了缓存组件实例的能力，类同 vue 框架中的 keep-alive 组件。react-router-cache-route 会根据路由匹配情况以及路由跳转方式缓存组件实例。基于 react-router 传递的数据，路由匹配情况可通过 props.match 判断；路由跳转方式可通过 props.history.action 判断。这些都在 CacheComponent 组件中实现。

CacheComponent 可通过 when 属性指定需缓存的路由跳转方式：

* forward：默认，通过 history.push、history.replace 跳转时需缓存
* back：通过 history.go 或浏览器的回退按钮等跳转时需缓存
* always：以上两种方式均需缓存

CacheComponent 会在路由匹配时将 state.cached 标识置为 true，即便 props.when 设置了 forward，实际路由采用 history.go 跳入。只有在跳出时，当且仅当为反向跳出时，CacheComponent 才会将 state.cached 标识置为 false。本文定义：

* 正向跳出：如 props.when 设置为 forward 时，使用 history.push、history.replace 跳出
* 反向跳出：如 props.when 设置为 forward 时，使用 history.go 或浏览器的回退按钮等跳出

正向跳出时，CacheComponent 内部组件实例会被缓存；反向跳出时，CacheComponent 会渲染 null，即清除内部组件实例缓存。CacheComponent 对于内部组件实例的缓存表现为：

1. 未缓存，匹配路由：使用随机的 key 键渲染组件实例，渲染时传入 didCache、didRecover 钩子
2. 正向跳出：使用 display: none 渲染组件实例，或 removeChild 移除组件实例并缓存，并调用内部组件的 didCache 钩子
3. 跳回匹配路由：使用缓存，移除组件实例的 display: none 样式，或使用 appendChild 渲染，并调用内部组件的 didRecover 钩子
4. 反向跳出：渲染 null
5. 跳回匹配路由：重新渲染组件实例

![image](url6.png)

因为 CacheComponent 在路由未匹配的时候也需要渲染，而 react-router 中的 Switch 组件会在路由未匹配时渲染 null，所以 react-router-cache-route 提供了 CacheSwitch、CacheRoute 等组件，即在路由未匹配时，也会渲染 CacheComponent。此外，CacheComponent 会缓存内部组件的滚动位置，以便使用缓存渲染时进行恢复。

页面组件的业务属性各异，使得在技术层面很难提供一套通用页面组件缓存框架加以处理。比如从列表页跳到详情页，详情页中更新的数据需要回显在列表页上，这时候就不能使用缓存；从列表页跳到另一个列表页，这时候就需要缓存。在使用 react-router-cache-route 制作页面级组件缓存的基础上，我们可以调用 didRecover 钩子更新缓存页面的数据。

### 研发框架中的前端路由

基于 webpack 的研发框架可通过制作入口文件的方式，使用 react-router 集成路由功能。基本的路由功能包含静态的配置式路由，动态的运行时路由，懒加载，路由守卫，页面切换动效等。

#### umi

umi 可以读取 .umirc.ts 配置文件中的路由配置信息或扫描 pages 文件夹，然后制作 react-router 类路由，即 .umi/router.ts 运行时文件中的内容。umi 也允许在 app.ts 文件中使用 [patchRoutes](https://umijs.org/zh-CN/docs/runtime-config#patchroutes-routes-) 接口追加路由、[onRouterChange](https://umijs.org/zh-CN/docs/runtime-config#onroutechange-routes-matchedroutes-location-action-) 监听路由变更。

#### remax

remax 对于 web 端应用，同样会制作应用的入口文件。它会使用 react-router-cache-route 缓存页面组件。结合 usePageEvent('onShow', () => {}) 生命周期，开发者可以更新页面级缓存组件的数据。因为 usePageEvent('onShow', () => {}) 生命周期会调用 didRecover 钩子。