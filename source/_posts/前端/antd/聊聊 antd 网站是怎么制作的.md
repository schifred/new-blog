---
title: 聊聊 antd 网站是怎么制作的
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 2bd74ebd
date: 2018-12-21 06:00:00
updated: 2018-12-21 06:00:00
---

翻看 [ant-design](https://github.com/ant-design/ant-design) 仓库，可以发现以下形式的 markdown 文档，这篇文章旨在于解答这样一个问题：ant design 是怎样把这些文档渲染成页面的。

```markdown
// ant-design Button 组件 demo
// 注释：元数据内容
---
order: 0
title:
  zh-CN: 按钮类型
  en-US: Type
---

// 注释：主体内容
## zh-CN

按钮有五种类型：主按钮、次按钮、虚线按钮、危险按钮和链接按钮。主按钮在同一个操作区域最多出现一次。

## en-US

There are `primary` button, `default` button, `dashed` button, `danger` button and `link` button in antd.

```jsx
import { Button } from 'antd';

ReactDOM.render(
  <div>
    <Button type="primary">Primary</Button>
    <Button>Default</Button>
    <Button type="dashed">Dashed</Button>
    <Button type="danger">Danger</Button>
    <Button type="link">Link</Button>
  </div>,
  mountNode,
);
```

在深入具体实现之前，先简要描述下 ant-design 的大致处理思路。为了将 markdown 文档渲染成静态网页，我们可以使用 webpack 加载器逐个加载 markdown 文件，在加载器中借助 [markdown-it](https://github.com/markdown-it/markdown-it) 解析这些文件（element-ui 就是基于 [markdown-it-chain](https://github.com/ulivz/markdown-it-chain) 实现的）（fusion 通过 node 服务在服务端使用 [remark](https://github.com/gnab/remark) 解析 markdown 文件）。与此不同的是，ant-design 并没有采用显式逐个加载 markdown 文件的方式，而是制作 entry 入口文件。入口文件使用 import 语句导入了占位用的 data.js 文件；在 data.js 文件加载过程中，ant-design 借助 webpack 加载器机制将指定目录的 markdown 文件解析成数据，作为 data.js 占位文件的加载内容。这一部分工作由 [bisheng](https://github.com/benjycui/bisheng) 完成。以下是 bisheng 启动开发服务器的主要流程。

![image](bisheng.png)

我们把 ant-design 网站的渲染机制分为两个部分：markdown 数据生成、markdown 数据渲染。

### markdown 数据生成

#### mark-twain

mark-twain 首先借助 yaml-front-matter 将 markdown 解析成 json 对象（该对象的 _content 属性为 markdown 中的主体内容），然后通过 remark 将 _content 内容解析成抽象语法树，最后由 mark-twain 的 转换器 将抽象语法树中的 root 等节点替换成可渲染的 html 节点。

yaml-front-matter 自身借助 js-yaml 实现。

```javascript
{
  "meta": {
    "order": 0,
    "title": {
      "zh-CN": "按钮类型",
      "en-US": "Type"
    }
  },
  "content":  [
    "article",
    ["h2", "zh-CN"],
    ["p", "按钮有五种类型：主按钮、次按钮、虚线按钮、危险按钮和链接按钮。主按钮在同一个操作区域最多出现一次。"],
    ["h2", "en-US"],
    ["p", "There are `primary` button, `default` button, `dashed` button, `danger` button and `link` button in antd."],
    [
      "pre",
      { "lang": "jsx" },
      { "code": "import { Button } from 'antd';..." }
  ]
}
```

#### bisheng

bisheng 在 mark-twain 的基础上，使用 child-process 启动多进程解析 markdown 文件，编译任务通过 node 进程的通信机制从主进程流入到子进程中，编译结果也通过 node 进程的通信机制从子进程带出到主进程中，详情可以参看 [boss.js](https://github.com/benjycui/bisheng/blob/master/packages/bisheng/src/loaders/common/boss.js)。实际的编译操作见于 [source-data.js](https://github.com/benjycui/bisheng/blob/master/packages/bisheng/src/utils/source-data.js) 文件，即如下：

```javascript
// process 方法会在子进程被使用
exports.process = (
  filename,// markdown 文件名
  fileContent,// markdown 文件内容
  plugins,// 插件
  transformers = [],// 转换函数，包含内置的 transformers/markdown（使用 mark-twain 编译）
  isBuild,// 是否生产环境
) => {
  let transformerIndex = -1;
  // 对于不同的文件使用不同的 transformer
  transformers.some(({ test }, index) => {
    transformerIndex = index;
    return eval(test).test(filename); // eslint-disable-line no-eval
  });
  const transformer = transformers[transformerIndex];

  // 编译
  const markdown = require(transformer.use)(filename, fileContent);

  // 使用插件进行处理，包含内置插件 bisheng-plugin-highlight 用于处理 jsx 等脚本
  // 以及 bisheng-plugin-description、bisheng-plugin-toc、bisheng-plugin-antd、bisheng-plugin-react
  const parsedMarkdown = plugins.reduce(
    (markdownData, plugin) =>
      require(plugin[0])(markdownData, plugin[1], isBuild === true),
    markdown,
  );

  return parsedMarkdown;
};
```

上文解释了 markdown 文件一旦被加载，它就会通过 process 方法被编译成 json 数据并输出。那么，markdown 文件是怎么被加载的呢？我们可以在 [updateWebpackConfig.js](https://github.com/benjycui/bisheng/blob/master/packages/bisheng/src/config/updateWebpackConfig.js) 文件中找到如下代码：

```javascript
webpackConfig.module.rules.push({
  test(filename) {
    return filename === path.join(bishengLib, 'utils', 'data.js') ||
      filename === path.join(bishengLib, 'utils', 'ssr-data.js');
  },
  loader: path.join(bishengLibLoaders, 'bisheng-data-loader'),
});
```

即在加载占位用的 data.js 文件时，bisheng 会使用内置的 [bisheng-data-loader](https://github.com/benjycui/bisheng/blob/master/packages/bisheng/src/loaders/bisheng-data-loader.js) 加载器加以处理。这个加载器的作用无他，就是扫描 bishengconfig 配置的 source 源文件夹下的 markdown 文件（bisheng 通过 [ramda](https://ramda.cn/) 将 markdown 文件解析成树形结构），并调用 boss.js 进行解析。这里干脆贴出附带说明的源码：

```javascript
module.exports = function bishengDataLoader(/* content */) {
  if (this.cacheable) {
    this.cacheable();
  }

  const { bishengConfig, themeConfig } = context;

  // markdown 全量文件
  const markdown = sourceData.generate(bishengConfig.source, bishengConfig.transformers);
  const browserPlugins = resolvePlugins(themeConfig.plugins, 'browser');
  const pluginsString = browserPlugins
    .map(plugin => `[require('${plugin[0]}'), ${JSON.stringify(plugin[1])}]`)
    .join(',\n');

  const callback = this.async();

  const picked = {};
  const pickedPromises = []; // Flag to remind loader that job is done.
  if (themeConfig.pick) {
    const nodePlugins = resolvePlugins(themeConfig.plugins, 'node');

    // 遍历 markdown 文件并作解析
    sourceData.traverse(markdown, (filename) => {
      const fileContent = fs.readFileSync(path.join(process.cwd(), filename)).toString();
      pickedPromises.push(new Promise((resolve) => {
        boss.queue({
          filename,
          content: fileContent,
          plugins: nodePlugins,
          transformers: bishengConfig.transformers,
          isBuild: context.isBuild,
          // 回调处理解析内容，包含 pick 分块
          callback(err, result) {
            const parsedMarkdown = eval(`(${result})`); // eslint-disable-line no-eval

            Object.keys(themeConfig.pick).forEach((key) => {
              if (!picked[key]) {
                picked[key] = [];
              }

              const picker = themeConfig.pick[key];
              const pickedData = picker(parsedMarkdown);
              if (pickedData) {
                picked[key].push(pickedData);
              }
            });

            resolve();
          },
        });
      }));
    });
  }

  // 将数据内容回写到 data.js 占位文件中
  Promise.all(pickedPromises)
    .then(() => {
      const sourceDataString = sourceData.stringify(markdown, {
        lazyLoad: themeConfig.lazyLoad,
      });
      callback(
        null,
        'module.exports = {' +
          `\n  markdown: ${sourceDataString},` +
          `\n  picked: ${JSON.stringify(picked, null, 2)},` +
          `\n  plugins: [\n${pluginsString}\n],` +
          '\n};',
      );
    });
};
```

### markdown 数据渲染

#### 路由信息

解析后的 markdown 数据会经由入口文件流入到路由信息获取函数中。入口文件负责直接调用路由信息获取函数，以供 react-router 使用。一般情况下，仅需要根据访问路径获取到对用的 markdown 数据，并使用配置约定的 React 组件进行渲染即可；特殊情况下，需要对流入组件的 props 数据进行转换（比如 Button 组件的 demo 数据就需要先添加到 props 中），这时会使用 collector 收集齐加以处理。

```javascript
// ant-design 中的路由配置
module.exports = {
  routes: {
    path: '/',
    component: './template/Layout/index',
    indexRoute: { component: homeTmpl },
    childRoutes: [
      {
        path: 'app-shell',
        component: appShellTmpl,
      },
      {
        path: 'index-cn',
        component: homeTmpl,
      },
      {
        path: 'docs/react/:children',
        component: contentTmpl,
      },
      {
        path: 'changelog',
        component: contentTmpl,
      },
      {
        path: 'changelog-cn',
        component: contentTmpl,
      },
      {
        path: 'components/:children/',
        component: contentTmpl,
      },
      {
        path: 'docs/spec/:children',
        component: contentTmpl,
      },
    ],
  }
}

// bisheng 中的路由信息获取函数，data 为解析后的 markdown 全量数据
function getRoutes(data) {
  // 根据路由配置中的渲染组件以及 location 中的路由参数获得渲染函数
  function templateWrapper(template, dataPath = '') {
    const Template = require(`{{ themePath }}/template${template.replace(/^\.\/template/, '')}`);

    return (nextState, callback) => {
      // 生成实际访问路径信息
      const propsPath = calcPropsPath(dataPath, nextState.params);
      // 从全量的 markdown 数据中获取访问路径对应的 markdown 数据
      const pageData = exist.get(data.markdown, propsPath.replace(/^\//, '').split('/'));
      // utils.exist 判断 markdown 数据是否包含指定 key 键的信息
      // utils.toReactComponent 根据 markdown 数据获得渲染组件
      const utils = generateUtils(data, nextState);

      // collector 用于处理 props
      const collector = (Template.default || Template).collector || defaultCollector;
      const dynamicPropsKey = nextState.location.pathname;
      const nextProps = {
        ...nextState,
        themeConfig,
        data: data.markdown,// 全量 markdown 数据
        picked: data.picked,// 分块的 markdown 数据
        pageData,
        utils,
      };
      collector(nextProps)
        .then((collectedValue) => {
          try {
            const Comp = Template.default || Template;
            Comp[dynamicPropsKey] = { ...nextProps, ...collectedValue };
            callback(null, Comp);
          } catch (e) { console.error(e) }
        })
        .catch((err) => {
          const Comp = NotFound.default || NotFound;
          Comp[dynamicPropsKey] = nextProps;
          callback(err === 404 ? null : err, Comp);
        });
    };
  }

  // 获取 theme 配置中的路由配置
  const themeRoutes = JSON.parse('{{ themeRoutes | safe }}');
  const routes = Array.isArray(themeRoutes) ? themeRoutes : [themeRoutes];

  // 基于路由配置生成 react-router 中可用的路由信息
  function processRoutes(route) {
    if (Array.isArray(route)) {
      return route.map(processRoutes);
    }

    return Object.assign({}, route, {
      onEnter: () => {
        if (typeof document !== 'undefined') {
          // 加载进度条
          NProgress.start();
        }
      },
      component: undefined,
      getComponent: templateWrapper(route.component, route.dataPath || route.path),
      indexRoute: route.indexRoute && Object.assign({}, route.indexRoute, {
        component: undefined,
        getComponent: templateWrapper(
          route.indexRoute.component,
          route.indexRoute.dataPath || route.indexRoute.path,
        ),
      }),
      childRoutes: route.childRoutes && route.childRoutes.map(processRoutes),
    });
  }

  const processedRoutes = processRoutes(routes);
  processedRoutes.push({
    path: '*',
    getComponents: templateWrapper('./template/NotFound'),
  });

  return processedRoutes;
};

// ant-design 中获取组件文档的 collector
collect(async nextProps => {
  const { pathname } = nextProps.location;
  // 访问路径
  const pageDataPath = pathname.replace('-cn', '').split('/');

  // 根据访问路径获取对应的 markdown 数据
  const pageData = isChangelog(pathname)
    ? nextProps.data.changelog.CHANGELOG
    : nextProps.utils.get(nextProps.data, pageDataPath);
  if (!pageData) {
    throw 404; // eslint-disable-line no-throw-literal
  }

  const locale = utils.isZhCN(pathname) ? 'zh-CN' : 'en-US';
  const pageDataPromise =
    typeof pageData === 'function'
      ? pageData()
      : (pageData[locale] || pageData.index[locale] || pageData.index)();
  const demosFetcher = nextProps.utils.get(nextProps.data, [...pageDataPath, 'demo']);
  // 获取组件 demo 数据
  if (demosFetcher) {
    const [localizedPageData, demos] = await Promise.all([pageDataPromise, demosFetcher()]);
    
    return { localizedPageData, demos };
  }
  return { localizedPageData: await pageDataPromise };
})(MainContent);
```

#### 组件面板

以下仅贴示 ant-design 组件的渲染类源码，不再详作介绍。想要在 codepen 中访问组件 demo，需要把特定数据内容提交到 https://codepen.io/pen/define；同样的，将特定数据提交到 https://codesandbox.io/api/v1/sandboxes/define，组件 demo 就能在 codesandbox 中访问了；通过使用 @stackblitz/sdk 能使组件 demo 在 stackblitz 中访问。在上述三个第三方应用访问组件 demo 的过程中，ant design 会使用 [Google Analytics](https://www.googletagmanager.com/gtag/js) 作记录。

```javascript
class ComponentDoc extends React.Component {
  render() {
    // ...
    // 组件 demo 处理
    showedDemo
      .sort((a, b) => a.meta.order - b.meta.order)
      .forEach((demoData, index) => {
        const demoElem = (
          <Demo
            {...demoData}
            key={demoData.meta.filename}
            utils={utils}
            expand={expandAll}
            location={location}
          />
        );
        if (index % 2 === 0 || isSingleCol) {
          leftChildren.push(demoElem);
        } else {
          rightChildren.push(demoElem);
        }
      });
    // ...

    return (
      { /* ... */ }
        <Row gutter={16}>
          { /* 左侧 demo 面板，或者一屏展示 demo */ }
          <Col
            span={isSingleCol ? 24 : 12}
            className={isSingleCol ? 'code-boxes-col-1-1' : 'code-boxes-col-2-1'}
          >
            {leftChildren}
          </Col>
          { /* 右侧 demo 面板 */ }
          {isSingleCol ? null : (
            <Col className="code-boxes-col-2-1" span={12}>
              {rightChildren}
            </Col>
          )}
        </Row>
      { /* ... */ }
    )
  }
}

class Demo extends React.Component {
  render() {
    // 组件 demo 文档经 bisheng 插件处理后会生成 preview 渲染函数，渲染函数注入 React, ReactDOM 完成渲染
    // 或组件 demo 文档直接输出 src 进行渲染
    if (!this.liveDemo) {
      this.liveDemo = meta.iframe ? (
        <BrowserFrame>
          <iframe src={src} height={meta.iframe} title="demo" />
        </BrowserFrame>
      ) : (
        preview(React, ReactDOM)
      );
    }

    return (
      { /* ... */ }
        { /* 组件 demo 渲染，ErrorBoundary 组件会使用 componentDidCatch 捕获错误 */ }
        <section className="code-box-demo">
          <ErrorBoundary>{this.liveDemo}</ErrorBoundary>
          {style ? (
            <style dangerouslySetInnerHTML={{ __html: style }} /> // eslint-disable-line
          ) : null}
        </section>
      { /* ... */ }
    )
  }
}
```

#### ssr 渲染

bisheng 会使用启动开发环境同样的流程制作入口文件以及入口 html 模板，与此同时，对于页面级的 markdown 文件，bisheng 会制作 ssr 函数基于 react-router 函数生成其他页面模板。以下是部分源码：

```javascript
// 打包函数
exports.build = function build(program, callback) {
  // ...

  webpack(webpackConfig, (err, stats) => {
    // 入口文件加载的 js、csss
    const manifest = getManifest(stats.compilation)[bishengConfig.entryName];

    // html 模板
    const template = fs.readFileSync(bishengConfig.htmlTemplate).toString();
    
    webpack(ssrWebpackConfig, (ssrBuildErr, ssrBuildStats) => {
      if (ssrBuildErr) throw ssrBuildErr;
      if (ssrBuildStats.hasErrors()) throw ssrBuildStats.toString('errors-only');

      require('./loaders/common/boss').jobDone();

      const { ssr } = require(path.join(tmpDirPath, `${entryName}-ssr`));
      const fileCreatedPromises = filesNeedCreated.map((file) => {
        const output = path.join(bishengConfig.output, file);
        mkdirp.sync(path.dirname(output));
        return new Promise((resolve) => {
          const pathname = filenameToUrl(file);
          ssr(pathname, (error, content, params = {}) => {
            if (error) {
              console.error(error);
              process.exit(1);
            }
            const templateData = Object.assign(
              {
                root: bishengConfig.root,
                content,
                hash: bishengConfig.hash,
                manifest,
                ...params,
              },
              bishengConfig.htmlTemplateExtraData || {},
            );
            const fileContent = nunjucks.renderString(template, templateData);
            fs.writeFileSync(output, fileContent);
            console.log('Created: ', output);
            resolve();
          });
        });
      });
      Promise.all(fileCreatedPromises).then(() => {
        if (callback) {
          callback();
        }
      });
    });
  });
};

function ssr(url, callback) {
  ReactRouter.match({ routes, location: url }, (error, redirectLocation, renderProps) => {
    if (error) {
      callback(error, '');
    } else if (redirectLocation) {
      callback(null, ''); // TODO
    } else if (renderProps) {
      const helmetContext = {};
      try {
        const content = ReactDOMServer.renderToString(
          <ReactRouter.RouterContext
            {...renderProps}
            createElement={(Component, props) => createElement(Component, { ...props, helmetContext })}
          />,
        );
        const helmet = helmetContext.helmet || Helmet.renderStatic();
        const documentTitle = DocumentTitle.rewind();
        const helmetTitleTmp = helmet.title.toString();
        const htmlAttributes = helmet.htmlAttributes.toString();
        const meta = helmet.meta.toString();
        const link = helmet.link.toString();
        const helmentTitle = helmetTitleTmp.match(/<title.*>([^<]+)<\/title>/)
          ? helmetTitleTmp.match(/<title.*>([^<]+)<\/title>/)[1]
          : '';

        // 兼容 DocumentTitle ，推荐使用 react-helmet
        const title = documentTitle || helmentTitle;
        // params for extension
        callback(null, content, {
          title,
          meta,
          link,
          htmlAttributes,
        });
      } catch (e) {
        callback(e, '');
      }
    } else {
      callback(null, ''); // TODO
    }
  });
};

function createElement(Component, props) {
  NProgress.done();
  const dynamicPropsKey = props.location.pathname;
  return <Component {...props} {...Component[dynamicPropsKey]} />;
};
```