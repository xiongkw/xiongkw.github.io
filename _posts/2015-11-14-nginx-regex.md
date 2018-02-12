---
layout: post
title: nginx 配置正则表达式写法
categories: [编程, nginx, web]
tags: [regex]
---

`nginx` 配置中正则表达式的定义，必须以下面四种组合开头：

* `~` 大小写不敏感匹配
* `~*` 大小写敏感匹配
* `!~` 大小写不敏感不匹配
* `!~*` 大小写敏感不匹配

