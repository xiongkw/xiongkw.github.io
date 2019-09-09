---
layout: post
title: Harbor高可用和镜像复制
categories: [编程, linux, docker]
tags: [harbor, keepalived]
---


> 通过`Keepalived`和镜像复制实现`Harbor`的高可用

#### 1. 部署架构

本方案使用`keepalived`做双机主备，实现`Harbor`的高可用

```
vip: 192.168.1.200
harbor1: 192.168.1.100
harbor2: 192.168.1.101
```

#### 2. 安装keepalived

参考[Keepalived安装]({{site.url}}/2015/11/20/keepalived-install)

#### 3. 安装harbor

参考[Harbor安装]({{site.url}}/2019/03/22/harbor-install)

#### 4. 配置harbor镜像复制

##### 4.1 新建复制目标

> 复制目标是指当前`Harbor`中的镜像需要复制到的目标`Harbor`

管理员选项-系统管理-新建目标

![]({{site.url}}/public/images/2019-03-26-harbor-keepalived-replication-1.png)

![]({{site.url}}/public/images/2019-03-26-harbor-keepalived-replication-2.png)

##### 4.2 新建复制策略

> 复制策略隶属于指定的`Harbor`项目，表示复制该项目下的所有镜像

项目详情-复制

![]({{site.url}}/public/images/2019-03-26-harbor-keepalived-replication-3.png)

![]({{site.url}}/public/images/2019-03-26-harbor-keepalived-replication-4.png)

##### 4.3 配置双向复制

用同样的方法配置从`Harbor101-library`到`Harbor100-libarary`的复制策略，即可实现两个`Harbor`之间的镜像同步

