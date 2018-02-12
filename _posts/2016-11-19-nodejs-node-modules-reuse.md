---
layout: post
title: Nodejs开发中node_modules复用技巧
categories: [编程, web]
tags: [nodejs, 前端]
---


> 使用`nodejs`做前端开发，每个`web`应用下都需要使用`npm install`安装前端`js`依赖，动辄数百M的`node_modules`文件如何重用？   
> 网上也有很多方法，比如设置环境变量`NODE_PATH`等，本文通过创建软链接实现`node_modules`复用

在`web`应用目录下创建软链接

`Linux`下
```
ln -s ~/share/node_modules node_modules
```

`Windows`下
```
mklink /h node_modules D:/work/share/node_modules
```