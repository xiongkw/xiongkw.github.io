---
layout: post
title: nginx server匹配规则
categories: [编程, nginx, web]
tags: []
---

nginx server匹配规则按优先顺序分为以下几种：

* 准确的server_name
* `*`开头的
* `*`结尾的
* 正则表达式
* server_name匹配不到的默认使用第一个server块

来个例子
```nginx
server{
    server_name xx.com;
    
}

server{
    server_name *.xx.com;
}

server{
    server_name www.xx.*;
}

server{
    server_name ~^(?<www>.+)\.xx\.com$;
}
```