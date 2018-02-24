---
layout: post
title: linux中使用free命令查看内存使用情况
categories: [编程, linux]
tags: [free]
---


> `linux`中通常使用`free`命令查看内存使用情况


例如：
```
[root@iZ94hu17gt5Z ~]# free -m
             total       used       free     shared    buffers     cached
Mem:          7567       7354        212         16         44       5721
-/+ buffers/cache:       1588       5978
Swap:            0          0          0

```

疑问：系统物理内存`8G，free`却只剩`212M`，程序不可能占用`7354M`这么多内存吧？

看下`free`命令的解释：
* `Mem`: 实际内存使用情况 `total = used + free`
* `-/+ buffers/cache`: 除去缓冲区和缓存, `程序实际使用内存 = used - buffers -cached`

> `linux`和`windows`对于内存使用的理念不同，`linux`认为资源就是用来使用的，不用白不用