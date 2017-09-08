---
layout: post
title: linux中使用free命令查看内存使用情况
categories: [编程, linux]
tags: [free]
---

> linux中使用free命令查看内存使用情况

```
[root@iZ94hu17gt5Z ~]# free -m
             total       used       free     shared    buffers     cached
Mem:          7567       7354        212         16         44       5721
-/+ buffers/cache:       1588       5978
Swap:            0          0          0

```

疑问：系统物理内存8G，free却只剩212M，程序不可能占用7354M这么多内存吧？

看下free命令的解释：
* Mem: 实际内存使用情况 total = used + free
* -/+ buffers/cache: 除去缓冲区和缓存 程序实际使用内存 = used - buffers -cached

总结：linux和windows对于内存使用的理念不同，linux认为资源就是用来使用的，不用白不用