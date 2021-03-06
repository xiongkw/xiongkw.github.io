---
layout: post
title: nginx server匹配规则
categories: [编程, nginx, web]
tags: []
---


#### 1. 匹配规则

`nginx server`匹配规则按优先顺序分为以下几种

* 准确的`server_name`
* `*`开头的
* `*`结尾的
* 正则表达式
* `server_name`匹配不到的默认使用第一个`server`块

#### 2. 一个例子

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

#### 3. 关于正则表达式

[nginx中正则表达式的写法]({{site.url}}/2015/11/14/nginx-regex/)