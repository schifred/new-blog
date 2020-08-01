---
title: MV* 架构
category:
  - 前端
  - 框架
tags:
  - 前端
  - 框架
keywords: 框架,MVC,MVVM
abbrlink: 10c7a78f
date: 2019-10-13 06:00:00
updated: 2019-10-13 06:00:00
---

### 前言

曾听人说起“前端没有架构”，我心里时常也有这种困惑。当框架、脚手架划定了前端代码的组织模式后，一方面使前端更聚焦于业务代码的编写，提升了工作时的投入产出比；另一方面也使前端更容易陷在琐碎的业务点上，压抑了饱含灵性的智力作业。但若仔细思索，我会认为前端也有其架构空间：比如设计复杂的业务模块、编辑器、工具库等；比如规划多应用构成的站点。一个人困于所见、所识、所遇，就会产生望洋兴叹、嗟伤时物的情绪。当前端从业者说“前端没有架构”的时候，也许是他的境界还没有达到一定高度，也许是他困于拧螺丝的活而没有出路。这篇文章写在分析 UForm 源码之前，目的在于逐步提炼对前端技术的系统化认知。介于水平的有限，这篇文章难免错谬。

### MVC 架构

MVC 架构源自桌面应用，在前后端领域均有见及。Model 为模型层，负责与外通信、对接数据；View 为视图层，负责构建视图内容；Controller 为控制器层，负责串联模型层和视图层。MVC 架构最常用见于单应用的组织形态。个人理解，在后端 Spring MVC 应用编码过程中，services 层可用于提供原子级的数据服务；controller 层可用于组合原子级的数据服务，并向页面模板注入内容；view 层以备模板引擎输出 html。controller 层在串联 service 和 view 层时，所采用的关键连接点是路由信息等。到了前端 SPA 应用中，路由仍旧是一个关键连接点，通过 controller 为不同页面输送数据。如果 controller 层进一步对接视图中的交互逻辑，那么 controller 层就与 Presenter 表现层表现相同，构成 MVP 架构（视图层和表现层双向通信）。本文并不区分 MVC 和 MVP 这两种架构。

与 jQuery 等 DOM 交互框架相比，采用 MVC 架构实现的应用和组件形态都更聚合，编码模式更为清晰，因此也更容易复用和维护。成熟的 MVC 框架一般通过事件监听或观察者模式实现，比如 backbone.js 等。

以下分别是采用 MVC 架构组织 SPA 应用、前端组件的简单设想，仅以说明 MVC 架构在前端中的应用形态。

#### 单页应用中的 MVC 架构

```javascript
/**
 * 引擎类，管理控制器、视图、模型
 */
class Engine {
  controllers: Controller[];
  views: {
    [key: string]: View
  };
  models: {
    [key: string]: Model
  };

  /* 注册视图 */
  registerView(name: string, view: View): void;

  /* 注册控制器 */
  registerController(controller: Controller): void;

  /* 注册模型 */
  registerModel(name: string, model: Model): void;

  /* 根据 hash 路由变更，调用控制器渲染视图 */
  onHashChange(): void;
}

/* 模型装饰器，用于为控制器类属性注入相应的 model，从引擎实例中获取 */
function model(name: string): Model;

/* 控制器装饰器，用于为视图类属性注入相应的 controllerl，从引擎实例中获取 */
function controller(name: string): Controller;

/**
 * 视图类，通过 @controller 为其属性装填特定的 controller
 */
class View {
  /* 页面模板 */
  tpl: string;

  /* 绑定事件，绑定函数可以是特定 controller 的方法 */
  binds(): void;

  /* 使用模板引擎渲染页面 */
  render(model): html;
}

/**
 * 控制器类，通过 @model 为其属性装填特定的 model
 */
class Controller {
  /* 路由 */
  route: string;

  /* 处理异步请求，变更模型数据，使用 view 渲染视图 */
  handler(): void;

  /* 仅举例视图交互逻辑的抽象 */
  onClick(event: any): void;
}

/**
 * 模型类
 */
class Model {
  /* 包含数据 */
  data: any;

  /* 仅举例异步请求的抽象 */
  get(params: Params): void;

  /* 仅举例业务处理逻辑的抽象 */
  add(item: any): void;
}
```

#### 前端组件中的 MVC 架构

```javascript
class Component {
  /* 渲染内容模板 */
  tpl: string;

  /* 数据模型 */
  model: any;

  /* 构建 dom 节点，并绑定事件 */
  view(): DOMElement;

  /* 将内容渲染到特定节点 */
  renderTo(data: any, elm: DOMElement): void;

  /* 控制器方法 */
  controllers: {
    [key: string]: Function,
  };
}
```

### MVVM 架构

MVVM 架构中的 VM 指的是 ViewModel，它实现了双向绑定，视图层的变更会自动通知到数据模型层，反之亦然。因此，MVVM 架构可以认为是一种自动化的 MVP 架构。典型的 MVVM 框架有 Vue, Angular 等。承接上文，以下是采用 MVVM 架构组织前端组件的简单设想。

```javascript
class ViewModel {
  /* 声明指令、绑定函数的渲染内容模板 */
  tpl: string;

  /* 数据模型 */
  data: any;

  /* 方法，可作为绑定函数 */
  methods: {
    [key: string]: Function,
  };

  /* 指令解析方式 */
  derectives: {
    [key: string]: Function,
  };

  /* 将模板解析成渲染函数，渲染函数以当前 ViewModel 为首参 */
  parse(): (vm: ViewModel) => DOMElement;

  /* 将内容渲染到特定节点 */
  render(data: any, elm: DOMElement): DOMElement;
}
```

典型的 tpl 模板如下：

```javascript
<div>
  <input type="text" v-model="newName"/>
  <p>{{newName}}</p>
</div>
```

解析 tpl 模板时，首先需要将节点建模成约定属性集合的树形节点；其次，在解析过程中，将模板中的标签内容转化为模型节点；然后，针对模型节点的特殊属性，通过可扩展的程式加以处理（如对于 v-model 指令，需要将 vm.data 数据与 input 节点进行双向绑定）；最后，通过递归下钻又回升的机制获得 dom 节点树，以供插入页面。我们就可以解释为什么 v-on 指令能将 vm.methods 能成为真实 dom 节点的绑定函数了。vm.data 数据又是怎么驱动视图重绘的呢？数据驱动视图更新的机制主要有手动调用、脏值检测、对象劫持、Proxy 代理。手动调用即抽象一般的数据变更方法，在该方法执行的尾部主动重新渲染组件（React 实现机制）。脏值检测即主动轮询树节点的数据属性，当发现其与 vm.data 数据不符时，即重新渲染组件（Angular 实现机制）。对象劫持即通过 Object.defineProperty 使数据变更的处理流程包含重绘组件这一过程（Vue 实现机制）。Proxy 与 Object.defineProperty 异曲同工（Mobx 实现机制）。组件绘制或重绘过程，可以使用虚拟 dom 提升效能，避免节点的全量渲染。这里仅点到为止，不再深入。想要了解更多，可以翻阅张成文的《现代前端技术解析》或各框架的相关文章。

### 后记

架构犹如轮廓，有助于理解成熟框架的设计和实现，不至于迷失在细节里。理想情况下，先定架构，再完善功能点也是一种高效的工作模式。现实中却不乏认知不到位、历史原因、项目工期短等问题，心态也需要有相当的韧性。