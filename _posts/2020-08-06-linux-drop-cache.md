---
layout: post
title: Linux清理缓存
categories: [编程, linux]
tags: [cache]
---

> 

#### 1. 查看内存使用情况

```
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           16G        15G        1G        4.0G        8G        8G
```

#### 2. drop cache

```
echo 1 > /proc/sys/vm/drop_caches // 表示清除pagecache。
echo 2 > /proc/sys/vm/drop_caches // 表示清除回收slab分配器中的对象（包括目录项缓存和inode缓存）。slab分配器是内核中管理内存的一种机制，其中很多缓存数据实现都是用的pagecache。
echo 3 > /proc/sys/vm/drop_caches // 表示清除pagecache和slab分配器中的缓存对象。
```

#### 3. 使用tee提高权限

```
$ echo 3 | sudo tee /proc/sys/vm/drop_caches
```