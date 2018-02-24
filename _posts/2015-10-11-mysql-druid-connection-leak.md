---
layout: post
title: Druid+mysql 连接泄露的问题
categories: [编程, mysql]
tags: [druid, leak]
---

> `Spring+Mybatis+Mysql`，使用`druid`作为`jdbc`连接池，在`druid`监控页面发现连接泄露

#### 1. 问题

```
逻辑连接打开次数	35601
逻辑连接关闭次数	34391

物理连接打开次数	1107
物理关闭数量	    394
```

所有`sql`操作都是由`mybatis mapper`完成，没有直接操作`Connection`的代码，所以代码问题不太可能

#### 2. `druid`配置

```properties
#初始连接数
initialSize=30
#最小连接数
minIdle=30
#最大连接数
maxActive=300

#Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接
timeBetweenEvictionRunsMillis=60000
#连接保持空闲而不被驱逐的最长时间
minEvictableIdleTimeMillis=300000

```

> `testWhileIdle`,发生在申请连接的时候，如果申请到的连接空闲时间大于`timeBetweenEvictionRunsMillis`，则执行`validationQuery`检测连接是否有效

#### 3. mysql配置

```properties
#连接空闲的最大时间(s)，超出后，mysql服务端会关闭连接
wait_timeout=300
```

#### 4. 原因

原因是`mysql`服务端主动关闭了连接，去掉`wait_timeout`参数(默认8h)，重启`mysql`和应用，连接关闭数正常了
