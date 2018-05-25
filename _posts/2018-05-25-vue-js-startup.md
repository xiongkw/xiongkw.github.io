---
layout: post
title: Vue.js开发环境搭建
categories: [编程, web]
tags: [vue, javascript, node, npm]
---


> 前端技术的发展真是日新月异，十年前我一次接触前端是在写`jsp`，写`freemarker velocity`，使用`jQuery`绑定事件操作`DOM`，到后来`前后端分离`和`单页面应用`，再到`angular react`，又到本文的`vuejs`

#### 1. 简介

> `Vue.js` 是用于构建交互式的 `Web` 界面的库，提供了 `MVVM` 数据绑定和一个可组合的组件系统，具有简单、灵活的 `API`

#### 2. 环境安装

##### 2.1 安装nodejs

下载并安装`nodejs`
[下载地址](http://nodejs.cn/download/)

查看`node`和`npm`版本
```
$ node -v
v10.1.0

$ npm -v
5.6.0
```

##### 2.2 安装vue-cli

通过`npm`安装`vue-cli`工具

```
npm install -g vue-cli
```

#### 3. 创建工程

##### 3.1 创建webpack工程

创建`webpack`模板工程

````
$ vue init webpack hello

(node:21432) ExperimentalWarning: The fs.promises API is experimental
? Project name (hello)
? Project name hello
? Project description (A Vue.js project)
? Project description A Vue.js project
? Author
? Author
? Vue build standalone
? Install vue-router? (Y/n)
? Install vue-router? Yes
? Use ESLint to lint your code? (Y/n)
? Use ESLint to lint your code? Yes
? Pick an ESLint preset (Use arrow keys)
? Pick an ESLint preset Standard
? Set up unit tests (Y/n) n
? Set up unit tests No
? Setup e2e tests with Nightwatch? (Y/n) n
? Setup e2e tests with Nightwatch? No
? Should we run `npm install` for you after the project has been created? (reco
? Should we run `npm install` for you after the project has been created? (reco
mmended) npm

   vue-cli · Generated "hello".


# Installing project dependencies ...
# ========================


> uglifyjs-webpack-plugin@0.4.6 postinstall D:\work\vue\hello\node_modules\webpack\node_modules\uglifyjs-webpack-plugin
> node lib/post_install.js

npm WARN rollback Rolling back node-pre-gyp@0.10.0 failed (this is probably harmless): EPERM: operation not permitted, rmdir 'D:\work\vue\hello\node_modules\fsevents\node_modules'
npm notice created a lockfile as package-lock.json. You should commit this file.
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.2.4 (node_modules\fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.4: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})

added 1225 packages in 193.186s


Running eslint --fix to comply with chosen preset rules...
# ========================


> hello@1.0.0 lint D:\work\vue\hello
> eslint --ext .js,.vue src "--fix"


# Project initialization finished!
# ========================

To get started:

  cd hello
  npm run dev

Documentation can be found at https://vuejs-templates.github.io/webpack
````

##### 3.2 工程结构

```
├── build/                        # webpack config files
│   └── ...
├── config/                       # main project config
│   ├── index.js
│   └── ...
├── dist/                         # where production build will go
├── src/
│   ├── api/                      # api
│   │   └── cart.js
│   ├── assets/                   # dynamic assets (processed by webpack)
│   ├── components/               # .vue components
│   ├── pages/                   
│   │   └── cart/                   
│   │       ├── *.module.js       # the spec module          
│   │       ├── *.vue             # .vue file     
│   │       └── *.router.js       # the router logic           
│   ├── store/
│   │   └── mutaition-types.js  
│   └── Main.js                   # entry js file
├── static/                       # pure static assets (directly copied)
├── .babelrc                      # babel config
├── .editorconfig                 # editor config
├── .eslintignore                 # ESlint ignore paths
├── .eslintrc.js                  # ESlint config
├── .gitignore                    # GIT ignore paths
├── .postcssrc.js                 # Postcss config
├── index.html                    # Main html template for webpack to inject deps
├── package.json                  # npm scripts and dependencies
└── README.md                     # readme for your App
```

##### 3.3 源码分析

通过运行命令`npm run dev`找到`script.dev`(位于`package.json`)
```json
 "scripts": {
    "dev": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
    "start": "npm run dev",
    "lint": "eslint --ext .js,.vue src",
    "build": "node build/build.js"
  },
```

> 关于[npm scripts](http://www.ruanyifeng.com/blog/2016/10/npm_scripts.html)

这里实际是调用`webpack-dev-server`启动一个`web`服务，使用的配置文件为`build/webpack.dev.conf.js`

```js
const utils = require('./utils')
const webpack = require('webpack')
const config = require('../config')
const merge = require('webpack-merge')
const path = require('path')
```

这里主要关注`config/index.js`
```js
dev: {
    assetsSubDirectory: 'static',
    assetsPublicPath: '/',
    proxyTable: {},

    // Various Dev Server settings
    host: 'localhost', // can be overwritten by process.env.HOST
    port: 8080, // can be overwritten by process.env.PORT, if port is in use, a free one will be determined
    autoOpenBrowser: false,
    errorOverlay: true,
    notifyOnErrors: true,
    poll: false, // https://webpack.js.org/configuration/dev-server/#devserver-watchoptions-
   }
```
主要的配置都在这里定义，可在这里修改开发环境的服务端口

再看`build` `node build/build.js`，使用的配置文件为`build/webpack.prod.conf.js`
```js
const path = require('path')
const utils = require('./utils')
const webpack = require('webpack')
const config = require('../config')
const merge = require('webpack-merge')
```

变量还是定义在`../config/index.js`
```js
build: {
    // Template for index.html
    index: path.resolve(__dirname, '../dist/index.html'),

    // Paths
    assetsRoot: path.resolve(__dirname, '../dist'),
    assetsSubDirectory: 'static',
    assetsPublicPath: '/',

  }
```

#### 4. 运行
执行命令
```
npm run dev
```

在浏览器访问
```
http://127.0.0.1:8080/
```

#### 5. 参考文档

* [vue](https://cn.vuejs.org/v2/guide/)

* [vue-cli](https://github.com/vuejs/vue-cli)

* [webpack模板](https://vuejs-templates.github.io/webpack/)

