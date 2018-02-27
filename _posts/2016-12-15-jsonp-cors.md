---
layout: post
title: jsonp和cors
categories: [编程, javascript]
tags: [jsonp, cors]
---


> 实现跨域资源访问的方法很多，这里介绍两种常用的`jsonp`和`cors`

#### 1. 关于同源策略

* `Cookie、LocalStorage 和 IndexDB`: 禁止对不同源的`Cookie、LocalStorage 和 IndexDB`操作
* `dom同源`：禁止对不同源页面`DOM`进行操作。这里主要场景是`iframe`跨域的情况，不同域名的`iframe`是限制互相访问的
* `ajax同源`：禁止`ajax`请求非同源的资源

> 所谓同源是指，`域名`，`协议`，`端口`相同

#### 2. jsonp
`Jsonp(JSON with Padding)` 是 `json` 的一种"使用模式"，可以让网页从别的域名（网站）那获取资料，即跨域读取数据。虽然`ajax`请求不可跨域，但是`script`标签本身是可以请求不同源资源的

以下是`www.a.com`域下的一个`a.html`
```html
<script>
function callback(data) {
  console.log(data);
}
</script>
<script src="http://www.b.com/data"/>
```

> 首先提供回调函数`callback`，然后使用`script`标签请求`www.b.com`下的`/data`资源   
> 这里可以发现`jsonp`只支持`GET`方法


以下是`http://www.b.com/data`返回结果：
```
callback({"id":"1","name":"fool"})
```

> 这里写死了回调函数为`callback`，当`a.html`中请求完成时便会执行`callback()`回调   
> [jQuery](http://api.jquery.com/jQuery.get/)中实现了通用的`jsonp`请求

#### 3. cors
`cors`: `Cross-origin resource sharing`, 是一个`W3C`标准，它允许浏览器向跨源服务器发出`ajax`请求

`cors`通过`http`响应头来实现：
```
Access-Control-Allow-Origin: http://www.b.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: xx

```

细节参考[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

#### 4. 参考

* [HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
* [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)