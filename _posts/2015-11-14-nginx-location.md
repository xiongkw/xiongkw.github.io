---
layout: post
title: nginx location匹配规则
categories: [编程, nginx, web]
tags: [location]
---

nginx location匹配规则按优先顺序分为以下几种：

* `=` 精确字符串匹配
* `^~` 最优普通字符串匹配
* `~[*]` 正则表达式匹配
* 字符串匹配

来个例子
```nginx
location ^~ /b/ {
    return 200 '^~ /b/'; 
}

location = /b/ {
    return 200 '= /b/';
}

location /b {
    return 200 '/b';
}

location ~ \.js$ {
        return 200 '~ \.js';
}

location ^~ /b/c/ {
    return 200 '^~ /b/c/'; 
}

location ~ \.html$ {
        return 200 '~ \.html';
}

location ~ \.htm.* {
        return 200 '~ \.htm.*';
}

loation / {
    return 200 '/';
}

```
测试一把：
```
请求              响应          说明
/b/              = /b/          精确匹配，只匹配/b/，优先级大于最优普通字符串
/b/c/           ^~ /b/c/        最优普通字符串匹配
/b/c/a.html     ^~ /b/c/        最优普通字符串匹配，优先级大于正则
/b/a.js         ^~ /b           最优普通字符串匹配，优先级大于正则
/a.js           ~ \.js          正则匹配
/a.html         ~ \.html        正则匹配，按定义顺序优先匹配
/a.htm          ~ \.htm.*       正则匹配
/b/c            /b              普通字符串匹配，匹配路径越长优先级越高
/a/             /               普通字符串匹配，匹配路径越短优先级越低
```
