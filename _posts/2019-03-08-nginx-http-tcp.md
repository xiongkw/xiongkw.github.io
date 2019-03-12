---
layout: post
title: nginx反向代理之http和tcp的比较
categories: [编程, nginx]
tags: [proxy, tcp]
---

> `nginx`从`1.9.0`开始支持四层代理了，一般来说四层代理的性能要优于七层，那么是否可以取代`http`代理呢

#### 1. 概述

* 目的：比较`http`和`stream`模式代理的性能

* 约束：单纯只比较代理性能，不考虑其它优化情况，例如`http`模式下的缓存等

#### 2. 测试方法

使用`ab`做压力测试，记录不同并发不同报文大小时的`rps`(每秒处理请求数)

* 主机资源: 三台`16核128G`物理机

* 部署结构: 两台主机分别部署`nginx`，一台主机充当客户机

```
nginx -> nginx
```

* 主要`nginx`配置

```
worker_processes  8;
events {
    worker_connections  65535;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    #keepalive_timeout  0;
    keepalive_timeout  65;

    upstream backend {
        server 192.168.1.61:8802;
        keepalive 1000;
    }

    server {
        listen       8802;
        server_name  localhost;
        proxy_buffering on;
        proxy_buffer_size 8k;
        proxy_buffers 8 32k;

        location / {
                proxy_pass http://backend;
                proxy_redirect off;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $host;
                proxy_http_version 1.1;
                proxy_set_header Connection "";
        }
    }
}

stream {
        upstream backend {
                server 192.168.1.61:8802;
        }

        server {
                listen 8803;
                proxy_pass backend;
                proxy_socket_keepalive on;
        }
}        
```

##### 2.1 测试backend

直接压测`backend`

| 响应大小 |   c100  |  c200 |  c500 | c1k | c2k | c5k | c10k |
| -------- | --------- | --------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| 10k | 27676 | 26466 | 25291 | 24839 | 22962 | 21267 | 21559 |
| 100k| 6352 | 6173 | 5613 | 5056| 4928 | 4969 | 4478 |

##### 2.2 http->backend

测试`http`模式代理`backend`的情况，测试命令：`ab -n 100000 -c 100 -k`

| 响应大小 |   c100  |  c200 |  c500 | c1k | c2k | c5k | c10k |
| -------- | --------- | --------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| 10k | 26511 | 24575 | 24577 | 21958 | 21719 | 22249 | 21910 |
| 100k| 4908 | 4736 | 4718 | 4595 | 4725 | 4509 | - |
    
> `nginx`到`upstream`配置了`keepalive`

##### 2.3 stream->backend

测试`stream`模式代理`backend`的情况，测试命令：`ab -n 100000 -c 100 -k`

| 响应大小 |   c100  |  c200 |  c500 | c1k | c2k | c5k | c10k |
| -------- | --------- | --------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| 10k | 26787 | 26278 | 25092 | 22906 | 22904 | 21251 | - |
| 100k| 4853 | 4638 | 4688 | 4760 | 4558 | 4487 | - |

##### 2.4 模拟高并发的情况

测试命令：`ab -n 100000 -c 100`

| 测试方法 | 响应大小 |   c100  |  c200 |  连接数(nginx->backend) |
| ------------- | -------- | --------- | --------- | ---------- |
| http | 10k | 5738 | 5479 | 1000 |
| http |100k| 1698 | 1936 | 1000 |
| stream |100k| 3708 | 3503 | 50000+ |
| stream |100k| 1465 | - | 50000+ |

#### 3. 分析

* 并发量不大时(本方案中小于`5000`时)，`stream`性能比`http`好
* 随着并发量和响应`body`逐渐增大，`http`代理的性能慢慢接近`tcp`，且此时`tcp`代理的稳定性降低
* 比较`nginx`到`backend`的`tcp`连接数，发现`http`一直稳定在`1000`，而`stream`则基本等于并发数，说明`http`模式重用了后端连接，而`stream`不重用
* 两种代理最大的区别在于是否复用`nginx`到`backend`的`tcp`连接，而实际对性能影响最大的因素在于`backend`是否能够支持与`nginx`入口相同的并发数，如果支持，则`tcp`方式性能更好，否则`http`方式因为能够复用底层`tcp`连接所以性能更好

* 结论：除去`http`代理的高级特性(例如`gzip，cache`等)之后，`stream`也无法完全取代`http`代理，`stream`适用于并发量不大的小报文长连接，`http`适用于并发量大的短连接，这正是传统`tcp`与`http`服务的应用场景区别
