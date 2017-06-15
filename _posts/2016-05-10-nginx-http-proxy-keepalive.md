---
layout: post
title: Nginx反向代理keepalive配置
categories: [编程]
tags: [nginx, 负载均衡, 反向代理, keepalive]
---


> 问题描述：使用nginx反向代理HTTP，在性能测试中发现tps还不及单个后端服务   
> 检查发现keepalive没有配置

```nginx
http
{
 
 upstream myapp.com {
  server 192.168.1.100:8080 weight=100;
  server 192.168.1.101:8080 weight=200;
  
  #keepalive超时时间s
  keepalive_timeout 60;
 }

 server
 {
  listen 80;
  server_name www.myapp.com;

  location / {
   proxy_pass http://myapp.com;
   #推荐他用http 1.1
   proxy_http_version 1.1;
   #Connection头信息默认为close，这里需要设置为空，表示不发送
   proxy_set_header Connection "";
   ...
 }

}
}

```
