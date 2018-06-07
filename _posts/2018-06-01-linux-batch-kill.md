---
layout: post
title: linux中使用awk和xargs批量kill进程
categories: [编程, linux]
tags: [awk, xargs]
---


> `linux`中杀死进程通常是先找出某个进程`id`，再使用`kill`命令

#### 1. 问题

需要`kill`一批进程，例如`apache storm`会启动`nimbus、supervisor、logviewer、ui`等进程，如果一个一个`kill`将会很麻烦

#### 2. 批量kill

使用`awk`和`xargs`命令批量`kill`进程

```
ps aux | grep storm | awk '{print $2}'| xargs kill -9 
```

#### 3. 解释

* `ps aux |grep storm`: 查找出包含`storm`的进程
* `awk '{print $2}'`: 打印进程ID
* `xargs kill -9`: 把`awk`命令中打印的进程`ID`传递给`kill`命令
