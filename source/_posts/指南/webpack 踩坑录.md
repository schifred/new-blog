---
title: webpack踩坑录
category:
  - 指南
tags:
  - 指南
  - webpack
keywords: 'webpack'
abbrlink: 3ba5be81
date: 2020-03-18 00:00:00
updated: 2020-03-18 00:00:00
---

### 懒加载

Q: 使用 webpack4, @babel/plugin-syntax-dynamic-import 插件后，react-router3 中基于 require.ensure 将不再生效，需要使用如 react-loadable 等加载方式。可参考 [react-router的3种按需加载介绍](https://blog.csdn.net/jacktesla/article/details/80792110)。
A: 猜想：webpack 中使用 require.ensure, import() 采用同一套机制，dynamic-import 插件也会影响 require.ensure，将导出模块转换成 promise。babel 文档 [@babel/plugin-syntax-dynamic-import](https://babeljs.io/docs/en/babel-plugin-syntax-dynamic-import/)、webpack 文档 [Dynamic Imports](https://webpack.js.org/guides/code-splitting/#dynamic-imports) 指明，import() 需要加载 [es6-promise](https://github.com/stefanpenner/es6-promise) 或者 [promise-polyfill](https://github.com/taylorhakes/promise-polyfill) 垫片。[require.ensure](https://webpack.js.org/api/module-methods/#requireensure) 是 webpack 所使用的传统方法。
