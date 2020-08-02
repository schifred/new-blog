---
title: single-spa 实现前端微服务
category:
  - 前端
  - 微服务
tags:
  - 前端
  - 微服务
keywords: 微服务
abbrlink: 9405b34d
date: 2020-02-25 06:00:00
updated: 2020-02-25 06:00:00
---

### 文档及示例

[single-spa](https://single-spa.js.org/) 具有以下功能特性：在无须刷新页面的前提下，同一个页面可使用不同的框架；基于不同框架实现的前端应用可以独立部署；制作新内容时可以使用不同的框架；支持应用内脚本的懒加载。

single-spa 借鉴了组件生命周期的思想，它为应用设置了针对路由的生命周期。当应用匹配路由/处于激活状态时，应用会把自身的内容挂载到页面上；反之则卸载。典型的 single-spa 由 html 页面、应用注册脚本、应用脚本自身构成。应用注册内容包含：name 应用名；loadingFunction 应用脚本加载函数；activityFunction 应用激活态判断函数。single-spa 又约定应用脚本包含以下生命周期：load 当应用匹配路由时就会加载脚本（非函数，只是一种状态）、bootstrap 引导函数（对接 html，应用内容首次挂载到页面前调用）、mount 挂载函数、unmount 卸载函数（须移除事件绑定等内容）、unload 非必要（unload 之后会重新启动 bootstrap 流程；借助 unload 可实现热更新）。生命周期函数获得参数包含 name 应用名、singleSpa 实例、mountParcel 手动挂载函数、customProps 自定义信息；它必须返回 Promise 或其本身为 async 函数；bootstrap、mount、unmount 生命周期函数不可缺省；生命周期函数可以指定多个，它们会构成异步调用链，逐个调用。官网中的实例如下：

```javascript
// 1. html 页面
<html>
<body>
  <script src="single-spa-config.js"></script>
</body>
</html>

// 2. 应用注册脚本
import * as singleSpa from 'single-spa';
const appName = 'app1';
const loadingFunction = () => import('./app1/app1.js');// loadingFunction，须返回 promise
const activityFunction = location => location.pathname.startsWith('/app1');// activityFunction，纯函数
const customProps = {};
singleSpa.registerApplication(appName, loadingFunction, activityFunction, customProps);// customProps 可以不填
singleSpa.start();

// 3. 应用脚本 app1.js，可以在另一个仓库中
let domEl;
export function bootstrap(props) {
  const {
    name,        // The name of the application
    singleSpa,   // The singleSpa instance
    mountParcel, // Function for manually mounting 
    customProps  // Additional custom information
  } = props;     // Props are given to every lifecycle
  return Promise
    .resolve()
    .then(() => {
      domEl = document.createElement('div');
      domEl.id = 'app1';
      document.body.appendChild(domEl);
    });
}
export function mount(props) {
  return Promise
    .resolve()
    .then(() => {
      domEl.textContent = 'App 1 is mounted!'
    });
}
export function unmount(props) {
  return Promise
    .resolve()
    .then(() => {
      domEl.textContent = '';
    })
}
```

#### 快速对接框架

在官方提供的示例项目 single-spa-examples 中，single-spa 提供了便捷的引导、挂载、卸载工具（如 single-spa-angular、single-spa-angularjs、single-spa-ember、single-spa-inferno、single-spa-preact、single-spa-react、single-spa-svelte、single-spa-vue），便于对接各种框架。具体可参考官方的示例项目或官方文档 Starting From Scratch。以下为 single-spa-react 的使用示例：

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import singleSpaReact from 'single-spa-react';
import App from './containers/App.js';
const reactLifecycles = singleSpaReact({
  React,
  ReactDOM,
  rootComponent: App,// 应用顶层组件
  domElementGetter,// 挂载应用的页面 dom 元素
});
export const bootstrap = [
  reactLifecycles.bootstrap,
];
export const mount = [
  reactLifecycles.mount,
];
export const unmount = [
  reactLifecycles.unmount,
];
function domElementGetter() {
  return document.getElementById("root");
}
```

#### 拆分部署

single-app 推荐的应用拆分部署策略有如下三种：

1. 将所有应用置于同一个包下。构建时间将会变慢，构建和部署都绑定在一起，这就需要固定的发布计划，而不是临时发布。
2. 创建根应用程序，以 npm 方式安装子应用。子应用程序有单独的存储仓库，每次更新时需要发版。根应用在单个 spa 应用更改时都需要重新安装、重建和重新部署。根应用和各个子应用也可以使用 monorepo 方法（借助 lerna 等）组织。
3. 创建根应用程序，以动态模块加载的方式加载子应用。独立发布的子应用提供一个活动 url 资源地址，随后在根应用中借助模块加载器（如 SystemJS）动态加载。

#### 超时

single-app 允许在应用脚本中设置超时时间，一旦超时，应用即予挂死。

```javascript
export function bootstrap(props) {...}
export function mount(props) {...}
export function unmount(props) {...}
export const timeouts = {
  bootstrap: {
    millis: 5000,
    dieOnTimeout: true,
  },
  mount: {
    millis: 5000,
    dieOnTimeout: false,
  },
  unmount: {
    millis: 5000,
    dieOnTimeout: true,
  },
  unload: {
    millis: 5000,
    dieOnTimeout: true,
  },
};
```

#### 动效

应用挂载、卸载时的动效既可以在应用内部处理，又可以使用 singlespa-transitions 处理。

#### 包

与应用不同，包没有 activityFunction 激活函数，而是通过手动调用加载。包可以像应用程序一样大，也可以像组件一样小。官方建议在应用程序上下文中装入包，那时包将与应用程序一起卸载。包有四种生命周期：bootstrap、mount、unmount、update。使用包允许我们在跨应用中共享组件。以下是官方示例：

```javascript
// 示例 1：普通应用中装入包
const parcelConfig = {
  bootstrap() {
    return Promise.resolve()
  },
  mount() {
    return Promise.resolve()
  },
  unmount() {
    return Promise.resolve()
  }
}

const domElement = document.getElementById('place-in-dom-to-mount-parcel')// 挂载点
const parcelProps = {domElement, customProp1: 'foo'}
const parcel = singleSpa.mountRootParcel(parcelConfig, parcelProps)
parcel.mountPromise.then(() => {// 包挂载后触发
  parcelProps.customProp1 = 'bar'
  return parcel.update(parcelProps)
})
.then(() => {
  return parcel.unmount()
})

// 示例 2：react 应用中装入包
import React from 'react'
import ReactDom from 'react-dom'
import singleSpaReact from 'single-spa-react'
import MyParcelComponent from './my-parcel-component.component.js'
export const MyParcel = singleSpaReact({
  React,
  ReactDom,
  rootComponent: MyParcelComponent
})

import Parcel from 'single-spa-react/parcel'
import MyParcel from './myparcel.js'
export class myComponent extends React.Component {
  render () {
    // 以 Parcel 组件形式加载包
    return (
      <Parcel
        config={MyParcel}
        { /* optional props */ }
        { /* and any extra props you want here */ }
      />
    )
  }
}

// 示例 3：跨应用复用包
// app1
export const AddContactParcel = {
  bootstrap: bootstrapFn,
  mount: mountFn,
  unmount: unmountFn,
}

// app2
componentDidMount() {
  SystemJS.import('App1').then(App1 => {
    const domElement = document.body
    App2MountProps.mountParcel(App1.AddContactParcel, {domElement})
  })
}
```

### 源码

在实现上，single-spa 约定了应用、包的生命周期，并调度着应用、包生命周期周转（路由匹配机制）的流程。至于应用、包的加载函数、路由匹配策略及其生命周期函数则由开发者手动实现。

#### 应用部分

single-app 将应用抽象为如下形式：

```javascript
{
  name: string;// 应用名
  parcels: {},// 包
  status: NOT_LOADED | LOADING_SOURCE_CODE | NOT_BOOTSTRAPPED | BOOTSTRAPPING | NOT_MOUNTED | 
    MOUNTING | UPDATING | LOAD_ERROR | MOUNTED | UNMOUNTING | SKIP_BECAUSE_BROKEN;// 应用状态
  customProps: { [key: string]: any };// 自定义属性
  loadImpl: string | (url: string) => Promise;// 应用模块位置或应用脚本加载函数
  activeWhen: (location: string) => boolean;// 应用激活状态判断函数
  bootstrap: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 引导函数
  mount: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 挂载函数
  unmount: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 卸载函数
  unload: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 卸载脚本函数
  timeouts: { bootstrap, mount, unmount, unload };// 超时设置
  loadErrorTime: null,// 
  devtools: { overlays },// 设置 window.__SINGLE_SPA_DEVTOOLS__ 的调试环境中使用
}
```

应用的生命周期（这些生命周期都会改变应用的状态）（如果设置了超时时间且 dieOnTimeout 为 true 时，bootstrap、mount、unmount 执行超时时会通过 Promise.reject 机制阻断后续流程）：

* load: 调用 app.loadImpl 加载脚本，并将 bootstrap、mount、unmount、unload 转化成异步调用链，设置超时时间。
* bootstrap: 调用 app.bootstrap 作引导处理。
* mount: 调用 app.mount 挂载内容，失败时 unmount。
* unmount: 首先对应用中的包执行 unmountThisParcel 方法，其次调用 app.unmount 卸载内容。
* unload: 调用 app.unload 卸载脚本，卸载后移除 app.bootstrap、app.mount、app.unmount、app.unload 等属性。unload 周期在开发者手动调用 unloadApplication 函数时触发，应用会重新退回到 NOT_LOADED 状态，需要再次执行 bootstrap 流程。

从全局层面看，single-app 按以下流程处理应用：

1. 全局劫持 hashchange、popstate 事件，绑定 reroute 函数；并捕获单页应用中的绑定函数，以备后续处理。
2. registerApplication(appName, applicationOrLoadingFn, activityFn, customProps) 注册应用。该过程将调用 reroute 函数加载应用脚本；其他使用 registerApplication 注册的应用将在等待状态，直到 reroute 递归调用时才予加载。备注：registerApplication 期间，single-app 会通过调用 ensureJQuerySupport 改写 $.on 绑定 hashchange、popstate 事件的功能，以便支持使用 window.$。
3. start() 启动应用，该过程也将调用 reroute 函数。start 若未执行，reroute 将只加载应用脚本，但不会调用应用脚本内的 bootstrap、mount、unmount 等生命周期函数。
4. 监听到 hashchange、popstate 事件，触发 reroute 函数卸载、挂载应用。对于挂载的应用，捕获到的绑定函数会在 reroute 尾端手动调用。

single-app 最核心的模块是 reroute，其负责调控应用脚本加载、卸载，应用内容挂载、卸载的流程。reroute 函数额外会使用 window.dispatchEvent 发送事件，以便于实现事件监听。

* 若应用未启动，通过 loadApps 加载匹配路由的应用脚本。加载完成后，若有其他应用脚本在排队注册中，递归调用 reroute 加载之。
* 若应用已启动，首先对不在用的应用脚本执行 unload、unmount 操作；然后对在用的但是未加载的应用脚本执行 load、bootstrap 操作，等到不在用的脚本 unmount 完成，再执行 mount 操作；其次对在用的同时已加载的应用脚本执行 bootstrap、mount 操作（mount 操作也要等到不在用的脚本 unmount 完成）。

```javascript
/**
 * reroute 或者由 registerApplication 触发调用，或者由 hashchange、popstate 事件触发调用。多个 registerApplication 会引起 reroute 递归调用；或者路由多次变更，reroute 未执行完成，也会导致 reroute 递归调动
 * reroute 作为编程接口，也可以后接 then 方法拿取已挂载的应用
 * @param {array} pendingPromises reroute 在执行期间，调用 registerApplication 或变更路由，pendingPromises 非空数组
 * @param {event} eventArguments 事件对象，hashchange、popstate 事件时携带
 * @state appChangeUnderway 指明 reroute 在一次执行周期中，其他 reroute 将被挂起，直到本次 reroute 递归调用才予以执行
 * @state wasNoOp 应用是否发生变更，如加载了一个应用，卸载了一个应用
 */
export function reroute(pendingPromises = [], eventArguments) {
  if (appChangeUnderway) {
    return new Promise((resolve, reject) => {
      peopleWaitingOnAppChange.push({
        resolve,
        reject,
        eventArguments,
      });
    });
  }

  appChangeUnderway = true;
  let wasNoOp = true;

  // single-app 是否已 start。start 意味应用变更由路由变更引起
  if (isStarted()) {
    return performAppChanges();
  } else {
    return loadApps();
  }

  // 筛选出待加载的脚本并加载之。所有加载完成后，调用 finishUpAndReturn
  function loadApps() {
    return Promise.resolve().then(() => {
      const loadPromises = getAppsToLoad().map(toLoadPromise);

      if (loadPromises.length > 0) {
        wasNoOp = false;
      }

      return Promise
        .all(loadPromises)
        .then(finishUpAndReturn)
        .catch(err => {
          callAllEventListeners();// 执行所有捕获的路由变更时间
          throw err;
        })
    })
  }

  // 处理路由变更引起的应用变更
  function performAppChanges() {
    return Promise.resolve().then(() => {
      window.dispatchEvent(new CustomEvent("single-spa:before-routing-event", getCustomEventDetail()));
      const unloadPromises = getAppsToUnload().map(toUnloadPromise);

      const unmountUnloadPromises = getAppsToUnmount()
        .map(toUnmountPromise)
        .map(unmountPromise => unmountPromise.then(toUnloadPromise));

      const allUnmountPromises = unmountUnloadPromises.concat(unloadPromises);
      if (allUnmountPromises.length > 0) {
        wasNoOp = false;
      }

      const unmountAllPromise = Promise.all(allUnmountPromises);

      // 先 load，再 bootstrap，然后等待其他应用 unmount、unload 完成，再 mount
      const appsToLoad = getAppsToLoad();
      const loadThenMountPromises = appsToLoad.map(app => {
        return toLoadPromise(app)
          .then(toBootstrapPromise)
          .then(app => {
            return unmountAllPromise
              .then(() => toMountPromise(app))
          })
      })
      if (loadThenMountPromises.length > 0) {
        wasNoOp = false;
      }

      // 已 bootstrap 过，再次 bootstrap，然后等待其他应用 unmount、unload 完成，再 mount
      const mountPromises = getAppsToMount()
        .filter(appToMount => appsToLoad.indexOf(appToMount) < 0)
        .map(appToMount => {
          return toBootstrapPromise(appToMount)
            .then(() => unmountAllPromise)
            .then(() => toMountPromise(appToMount))
        })
      if (mountPromises.length > 0) {
        wasNoOp = false;
      }

      return unmountAllPromise
        .catch(err => {
          callAllEventListeners();
          throw err;
        })
        .then(() => {
          // 非必须的应用已卸载，可以安心调用挂载应用的 hashchange、popstate 绑定函数
          callAllEventListeners();

          return Promise
            .all(loadThenMountPromises.concat(mountPromises))
            .catch(err => {
              pendingPromises.forEach(promise => promise.reject(err));
              throw err;
            })
            .then(() => finishUpAndReturn(false))
        })
    })
  }

  function finishUpAndReturn(callEventListeners=true) {
    const returnValue = getMountedApps();

    if (callEventListeners) {
      callAllEventListeners();
    }
    pendingPromises.forEach(promise => promise.resolve(returnValue));

    try {
      const appChangeEventName = wasNoOp ? "single-spa:no-app-change": "single-spa:app-change";
      window.dispatchEvent(new CustomEvent(appChangeEventName, getCustomEventDetail()));
      window.dispatchEvent(new CustomEvent("single-spa:routing-event", getCustomEventDetail()));
    } catch (err) {
      // single-spa:no-app-change 事件监听器报错，single-spa 不予处理
      setTimeout(() => {
        throw err;
      });
    }

    // reroute 单次执行结束，无挂起的 reroute 时，调用 reroute 将直接执行
    appChangeUnderway = false;

    // 被挂起的 reroute，需要手动触发之
    if (peopleWaitingOnAppChange.length > 0) {
      const nextPendingPromises = peopleWaitingOnAppChange;
      peopleWaitingOnAppChange = [];
      reroute(nextPendingPromises);
    }

    return returnValue;
  }

  // 手动调用挂载应用中被阻断的 haschange、popstate 绑定函数
  function callAllEventListeners() {
    pendingPromises.forEach(pendingPromise => {
      callCapturedEventListeners(pendingPromise.eventArguments);
    });

    callCapturedEventListeners(eventArguments);
  }

  function getCustomEventDetail() {
    const result = {detail: {}}

    if (eventArguments && eventArguments[0]) {
      result.detail.originalEvent = eventArguments[0]
    }

    return result
  }
}
```

#### 包部分

single-app 将包抽象为如下内部表现形式：

```javascript
{
  id: number;// id
  name: string;// 包名
  parentName: string;// 父包名或应用名
  parcels: {},// 包，以 { id: parcel } 形式存储
  status: NOT_LOADED | LOADING_SOURCE_CODE | NOT_BOOTSTRAPPED | BOOTSTRAPPING | NOT_MOUNTED | 
    MOUNTING | UPDATING | LOAD_ERROR | MOUNTED | UNMOUNTING | SKIP_BECAUSE_BROKEN;// 状态
  customProps: { [key: string]: any };// 自定义属性，包含 domElement 挂载节点
  bootstrap: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 引导函数
  mount: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 挂载函数
  update: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 更新函数
  unmountThisParcel: () => Promise;// 卸载接口，内部会调用 parcel.unmount 方法
  unmount: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 卸载函数
  unload: (props: { customProps, name, mountParcel, singleSpa }) => Promise;// 卸载脚本函数
  timeouts: { bootstrap, mount, unmount, unload };// 超时设置
}
```

singleSpa.mountRootParcel(config, customProps)、singleSpa.mountParcel(config, customProps) 方法能将外部配置项 [config.name]、config.bootstrap、config.mount、[config.update]、config.unmount、customProps.domElement 转化成内部表现形式（config 可以包模块的加载函数）。在 singleSpa.mountRootParcel、singleSpa.mountParcel 调用期间，single-spa 会加载包模块，并为内部表现形式添加 name、bootstrap、mount、update、unmount、timeouts 等属性，并返回 { mount, unmount, getStatus, loadPromise, bootstrapPromise, mountPromise, unmountPromise } 对象，用于手动挂载或卸载包。

特别注意，singleSpa.mountRootParcel 将包挂在顶部；singleSpa.mountParcel 一般以应用的 props.mountParcel 形式使用，也即作为应用下的包，其将会随着应用的销毁而销毁。

从总体层面看，包加载、卸载的机制与应用相同，只是多了一个 update 生命周期，以及包的存活空间限制、包需要手动渲染。

#### single-spa-react

[single-spa-react](https://github.com/single-spa/single-spa-react) 便于 react 应用快速对接 single-app。

singleSpaReact(opts) 函数能指定顶层组件及其渲染位置、渲染方式，其返回内容可作为应用或包脚本的 bootstrap、mount、unmount、update 导出。

```javascript
singleSpaReact(opts: { 
  React: React;
  ReactDOM: ReactDOM;
  rootComponent: React.Component;// 顶层组件，通过 React.createElement 加载元素
  loadRootComponent: () => React.Component;// 顶层组件加载函数，与 rootComponent 选填一项即可
  suppressComponentDidCatchWarning: boolean;// react16 以上版本，若 rootComponent 未实现 componentDidCatch 方法，予以警告
  domElementGetter?: () => DOMElement;// 获取挂载节点，不填会创建 id 为 `single-spa-application:${appName}` 的节点进行挂载
  parcelCanUpdate: boolean;// 包是否可渲染
  renderType?: 'createRoot' | 'createBlockingRoot' | 'hydrate';// 渲染方式，分别使用 ReactDOM.createRoot、ReactDOM.createBlockingRoot、ReactDOM.hydrate、ReactDOM.render 方法渲染
}): {
  bootstrap: (props) => Promise;// 加载顶层组件脚本
  mount: (props) => Promise;// 将顶层组件渲染到页面上
  unmount: (props) => Promise;// 使用 ReactDOM.unmountComponentAtNode 移除顶层组件
  update: (props) => Promise;// 指定 parcelCanUpdate 前提下，刷新 rootComponent
};
```

特别的，当以 React.createContext 机制赋值 SingleSpaContext 导出时，single-spa-react 会为所有应用、包添加上下文容器 SingleSpaContext.Providev，其 value 值为 mount 生命周期的 props（包含 mountParcel 方法，指定在当前应用或包中渲染其他包）。

single-spa-react 额外提供了 Parcel 组件，其接受 props.config 为包的生命周期导出模块（即 singleSpaReact 函数返回值），默认会将包渲染到组件树中；当指定 props.appendTo 属性，则会将包渲染到 props.appendTo 元素上。渲染的前提是它能获得 SingleSpaContext 传递的 mountParcel 函数（即包挂载在应用或其他包的 rootComponent 下），否则需要开发者显式指定 props.mountParcel 方法。

### 后记

造物必有迹可循，只是曲折的历史会把它打扮得较难领会。创造者不只创造为外部所用的产物，还有对内部加倍透明的历史。如果只是使用，不是创造，何必知晓历史之古。