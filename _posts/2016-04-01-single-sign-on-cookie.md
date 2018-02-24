---
layout: post
title: 基于cookie共享实现单点登录
categories: [编程, web]
tags: [http, sso, cookie, session]
---


> 基于`cookie`共享实现简单的单点登录

#### 1. 背景
`web`服务采用`session`跟踪用户和服务器的交互信息，由于`HTTP`请求是无状态的，所以采用`cookie`记录用户身份，实现会话跟踪

> `cookie`在会话中通常用于记录用户会话id，例如：`JSESSIONID=xxx`

#### 2. cookie的几个重要属性
* `Max-Age`: 生存周期单位s，默认为`-1`表示浏览器关闭即销毁
* `domain`: 所属域，相同域和其子域下可以共享，例如`abc.com、a.abc.com`和`b.abc.com`都可以访问`abc.com`下的`cookie`
* `path`: 所属路径，路径匹配的url可以共享，例如`/myapp/xx、/myapp/bb/a.html`和`/myapp/`都可以访问路径为`/myapp/`的`cookie`
* `secure`:当设置为`true`时，表示创建的 `Cookie` 会被以安全的形式向服务器传输，也就是只能在 `HTTPS` 连接中被浏览器传递到服务器端进行会话验证，如果是 `HTTP` 连接则不会传递该信息，所以不会被窃取到`Cookie` 的具体内容
* `HttpOnly`:如果在`Cookie`中设置了`"HttpOnly"`属性，那么通过程序(`JS`脚本、`Applet`等)将无法读取到`Cookie`信息，这样能有效的防止`XSS`攻击

#### 3. 实例
通过设置两个应用`cookie`的`domain`为相同的顶级域名实现`cookie`共享
```
应用域名        domain     path
app1.myapp.com  myapp.com   /
app2.myapp.com  myapp.com   /
```