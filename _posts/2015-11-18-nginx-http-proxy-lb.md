---
layout: post
title: nginx反向代理负载算法
categories: [编程, nginx, web]
tags: [http, 负载均衡, 反向代理]
---

#### 1.轮询
默认的负载算法，逐一分配
```nginx
 upstream myapp.com {
  server 192.168.1.100:8080;
  server 192.168.1.101:8080;
 }

```

#### 2.加权轮询
按权重轮询，权重大的分配到的几率就大
```nginx
 upstream myapp.com {
  server 192.168.1.100:8080 weight=100;
  server 192.168.1.101:8080 weight=200;
 }

```

#### 3.ip_hash
相同的`client`端分配到相同的后端服务，通常用于实现`会话粘贴`
```nginx
 upstream myapp.com {
  ip_hash;
  server 192.168.1.100:8080;
  server 192.168.1.101:8080;
 }

```
#### 4.fair
按后端服务器性能分配，性能更好后端会分配到更多的请求
```nginx
 upstream myapp.com {
  server 192.168.1.100:8080;
  server 192.168.1.101:8080;
  fair;
 }

```
#### 5.url_hash
相同的`url`分配到相同的后端服务，后端服务为缓存时比较有效
```nginx
 upstream myapp.com {
  server 192.168.1.100:8080;
  server 192.168.1.101:8080;
  hash $request_uri;  
  hash_method crc32;
 }

```