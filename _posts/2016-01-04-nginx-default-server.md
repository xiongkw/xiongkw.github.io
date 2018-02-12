---
layout: post
title: Nginx default_server的配置
categories: [编程, nginx, web]
tags: [server]
---

> `nginx` 配置中配置多个`server`时，访问一个不存在的`server_name`也会默认匹配到第一个`server`块

```nginx
server{
    server_name xx.com;
    
}

```

解决办法：定义一个默认的`server`块
```nginx
server{
	listen      80 default_server;
	server_name "";
	return      444;
}
```

> `Nginx`官方解释：   
> Here, the server name is set to an empty string that will match requests without the “Host” header field, and a special nginx’s non-standard code 444 is returned that closes the connection.