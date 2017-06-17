---
layout: post
title: Nginx反向代理Cookie支持
categories: [编程, web, nginx]
tags: [http, nginx, 负载均衡, 反向代理, cookie, session]
---


> 问题描述：使用nginx反向代理HTTP，发现session获取不到   
> 原因是http转发时没有带上cookie

```nginx
  location / {
   proxy_pass http://myapp.com;
   
   #转发时带上cookie头信息
   proxy_set_header Cookie $http_cookie;
   
   # 替换cookie domain，常用于cookie跨域共享
   proxy_cookie_domain myapp.com nginx.server;
   
   # 替换cookie path，常用于cookie跨域共享
   proxy_cookie_path /myapp/ /;
   
   ...
 }

```
