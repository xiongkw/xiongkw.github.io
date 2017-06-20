---
layout: post
title: Nginx default_server的配置
categories: [编程, web, nginx]
tags: [nginx, web, server]
---

> nginx 配置中配置多个server时，访问一个不存在的server_name也会默认匹配到第一个server块

```nginx
server{
    server_name xx.com;
    
}

```

解决办法：定义一个默认的server块
```nginx
server{
	listen      80 default_server;
	server_name "";
	return      444;
}
```

> Nginx官方解释：   
> Here, the server name is set to an empty string that will match requests without the “Host” header field, and a special nginx’s non-standard code 444 is returned that closes the connection.