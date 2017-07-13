---
layout: post
title: web.xml中servlet和filter的url-pattern配置和匹配规则
categories: [编程, java, web]
tags: [java, web, servlet, filter, url-pattern]
---

web.xml中`servlet`的`url-pattern`一般有四种写法：

1.固定字符串
```
/a/b/c
```
> 精确匹配，优先级最高

2.路径通配
```
/a/*
```
> 路径匹配，优先级仅次于精确匹配。匹配到多个时，匹配路径越长优先级越高。

3.扩展名匹配
```
*.do
```
> 优先级次于路径匹配

4.默认匹配
```
/
```
> 固定写法，优先级最低

`filter`中`url-pattern`的定义同`servlet`，但是匹配规则不同。所有匹配的`filter`都会拦截到请求，先后顺序由`filter`在web.xml中定义的顺序决定。