---
layout: post
title: nginx反向代理tcp时连接超时的问题
categories: [编程, nginx]
tags: [proxy, tcp]
---

> `nginx`反向代理`TCP`，一段时间(10分钟)后客户端连接超时，而客户端直连服务端则不会出现此问题

#### 1. nginx配置

```
worker_processes 8;
worker_rlimit_nofile 655350;
events {
	use epoll;
	worker_connections 1024;
}

stream {
	log_format basic '$remote_addr:$remote_port [$time_local] $ssl_preread_server_name $protocol $status $bytes_sent $bytes_received $connection $session_time';
	access_log /app/nginx/logs/access.log basic buffer=32k;
	server {
		listen	8000;
		preread_timeout 60s;
		proxy_buffer_size 2m;
		proxy_connect_timeout 60s;
		proxy_socket_keepalive on;
		ssl_preread on;
		proxy_pass $ssl_preread_server_name;
	}
}
```

#### 2. 日志

查看access日志

```
192.168.1.101:5080 [24/Jun/2019:10:38:14 +0800] 192.168.1.100:7006 TCP 200 2687 1767 285 600.150
192.168.1.101:5076 [24/Jun/2019:10:38:14 +0800] 192.168.1.100:7006 TCP 200 2687 1767 280 600.153
192.168.1.101:5073 [24/Jun/2019:10:38:14 +0800] 192.168.1.100:7006 TCP 200 2687 1767 288 600.150
192.168.1.101:5077 [24/Jun/2019:10:38:14 +0800] 192.168.1.100:7006 TCP 200 2687 1767 279 600.155
192.168.1.101:5078 [24/Jun/2019:10:38:14 +0800] 192.168.1.100:7006 TCP 200 2687 1767 293 600.147
192.168.1.101:5079 [24/Jun/2019:10:38:14 +0800] 192.168.1.100:7006 TCP 200 2687 1767 282 600.152
```

发现会话时间都在`600s`

#### 3. 分析

查看nginx参数`proxy_timeout`文档

```
Syntax:	proxy_timeout timeout;
Default: proxy_timeout 10m;
Context: stream, server
```

> Sets the timeout between two successive read or write operations on client or proxied server connections. If no data is transmitted within this time, the connection is closed.

该参数的作用是维护nginx到客户端和nginx到代理服务的tcp长连接，默认是10m，这也解释了为什么客户端10分钟后会断开

#### 4. 调整proxy_timeout

修改`proxy_timeout`参数为1小时，客户端不会出现连接超时了

```
proxy_timeout 60m;
```

#### 5. 客户端心跳

查看客户端，发现客户端每隔15分钟会发送一次心跳，所以`proxy_time`的值需要大于15m才有效

#### 6. 总结

虽然nginx提供了长连接的方法，但还是建议由客户端来保持长连接，因为在网络中的每一层，比如防火墙、LVS、nginx等都有自己的keepalive算法，而且这些都是不可控的

#### 7. 参考

* [proxy_timeout](http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html#proxy_timeout)
* [listen ](http://nginx.org/en/docs/stream/ngx_stream_core_module.html#listen)