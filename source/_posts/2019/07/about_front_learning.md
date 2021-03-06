---
layout: "post"
title: "前端入门文档索引"
date: "2019-07-12"
categories: 前端
---

# 基础知识

前端开发的基础知识可以在 **MDN Web Docs** 获得，主要是 [HTML](https://developer.mozilla.org/zh-CN/docs/Web/HTML)，[CSS](https://developer.mozilla.org/zh-CN/docs/Web/CSS)，[JavaScript](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)。
<!-- more-->

# 语言相关

JavaScript 语言本身在不断演进，一个比较重要的版本更新是 EcmaScript 2015 （即 ES6），具体可参阅以下文档：
- [ECMAScript 6 入门](http://es6.ruanyifeng.com)
- [es6-cheatsheet](https://github.com/DrkSephy/es6-cheatsheet)

JavaScript 本身没有类型检查，因此微软推出了它的严格超集 TypeScript，提供静态类型检查，具体可参阅以下文档：
- [官方手册](http://www.typescriptlang.org/docs/handbook/basic-types.html)
- [官方手册中文版](https://www.tslang.cn/docs/handbook/basic-types.html)
- [TypeScript Deep Dive](https://basarat.gitbooks.io/typescript)

# React 相关

React 是 Facebook 推出的前端 UI 开发框架，具体可参考以下资料：
- [官方文档](https://reactjs.org/docs/getting-started.html)
- [官方文档中文版](https://zh-hans.reactjs.org/docs/getting-started.html)
- [React 入门实例教程](http://www.ruanyifeng.com/blog/2015/03/react.html)

在前端路由方面，有比较多的 React 库可供选择，比较流行的是 React Router，可参考：
- [React Router 官方文档](https://reacttraining.com/react-router/web/guides/quick-start)
- [React Router 4 简易入门](https://segmentfault.com/a/1190000010174260)

# 全局状态管理

当 app 状态共享及变化很复杂的时候需要使用全局状态管理的库，比较流行的是 Redux：
- [Redux 官方文档](https://redux.js.org/introduction/getting-started)
- [Redux 中文文档](http://cn.redux.js.org/index.html)

Redux 可用于不同的前端框架，在 React 项目里使用需要用到相关的绑定库：
- [react-redux 官方文档](https://react-redux.js.org/introduction/quick-start)
- [react-redux 官方文档中文版](https://www.redux.org.cn/docs/react-redux)

Redux 本身不处理异步操作及副作用，需要集成其他库，主要有以下几个：
- [redux-thunk](https://github.com/reduxjs/redux-thunk)
- [redux-saga](https://github.com/redux-saga/redux-saga)
- [redux-observable](https://github.com/redux-observable/redux-observable)

# 综合相关

[TypeScript + React + Redux 项目脚手架](https://github.com/Microsoft/TypeScript-React-Starter#typescript-react-starter)