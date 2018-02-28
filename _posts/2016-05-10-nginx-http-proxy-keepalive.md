---
layout: post
title: nginx反向代理keepalive配置
categories: [编程, nginx, web]
tags: [负载均衡, 反向代理, keepalive, http]
---


> 使用`nginx`反向代理`HTTP`，在性能测试中发现`tps`还不及单个后端服务   

#### 1. 原因

通过网络状态看到`nginx`对后端服务的访问连接频繁创建和销毁，原因在于`nginx`访问后端服务使用的是`http`短连接

#### 2. 配置长连接

```
http
{
 #一个长连接上可以服务的请求的最大数，默认100，超出后连接将被关闭
 keepalive_requests 100000;
 #keepalive超时时间s
 keepalive_timeout 60s;
 
 upstream myapp.com {
  server 192.168.1.100:8080 weight=100;
  server 192.168.1.101:8080 weight=200;
  
  #一个worker进程和后端server保持长连接的最大数，超出后将会关闭最久未使用的连接
  keepalive 10000;
 }

 server
 {
  listen 80;
  server_name www.myapp.com;

  location / {
   proxy_pass http://myapp.com;
   #推荐使用http 1.1
   proxy_http_version 1.1;
   #Connection头信息默认为close，这里需要设置为空，表示不发送
   proxy_set_header Connection "";
   ...
 }

}
}

```
