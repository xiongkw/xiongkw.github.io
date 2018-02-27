---
layout: post
title: nginx搭建HTTP反向代理
categories: [编程, nginx, web]
tags: [http, 负载均衡, 反向代理]
---

> 使用`nginx`搭建`HTTP`反向代理

#### 一个例子

```nginx
worker_processes 8;
worker_rlimit_nofile 65535;

events
{
 use epoll;
 worker_connections 65535;
}

http
{
 upstream myapp.com {
  server 192.168.1.100:8080 weight=100;
  server 192.168.1.101:8080 weight=200;
 }

 server
 {
  listen 80;
  server_name www.myapp.com;

  location / {
   proxy_pass http://myapp.com;
   proxy_redirect off;
   proxy_set_header X-Real-IP $remote_addr;
   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   proxy_set_header Host $host;
 }

}
}

```

* `upstream`: 代理服务器
* `proxy_pass`: 反向代理