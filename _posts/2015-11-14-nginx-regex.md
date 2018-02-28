---
layout: post
title: nginx中正则表达式的写法
categories: [编程, nginx, web]
tags: [regex]
---

#### 1. nginx中正则表达式的写法

`nginx` 配置中正则表达式的定义，必须以下面四种组合开头：

* `~` 大小写不敏感匹配
* `~*` 大小写敏感匹配
* `!~` 大小写不敏感不匹配
* `!~*` 大小写敏感不匹配

#### 2. nginx中正则表达式的应用

* [nginx server匹配规则]({{site.url}}/2015/11/13/nginx-server/)
* [nginx location匹配规则]({{site.url}}/2015/11/14/nginx-location/)