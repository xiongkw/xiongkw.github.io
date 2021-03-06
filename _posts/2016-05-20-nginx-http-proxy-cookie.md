---
layout: post
title: nginx反向代理Cookie支持
categories: [编程, nginx, web]
tags: [http, 负载均衡, 反向代理, cookie, session]
---


> 使用`nginx`反向代理`HTTP`，发现`session`获取不到   

#### 1. 原因

通过服务日志看到`http`请求中的`cookie`信息丢失了，原因是被`nginx`丢弃了

#### 2. 配置cookie转发

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
