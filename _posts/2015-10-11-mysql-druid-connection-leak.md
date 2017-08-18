---
layout: post
title: Druid+mysql 连接泄露的问题
categories: [编程, mysql]
tags: [mysql, druid]
---

> `Spring+Mybatis+Mysql`，使用druid作为jdbc连接池，在druid监控页面发现连接泄露

```
逻辑连接打开次数	35601
逻辑连接关闭次数	34391

物理连接打开次数	1107
物理关闭数量	    394
```

所有sql操作都是由mybatis mapper完成，没有直接操作Connection的代码，所以代码问题不太可能

检查druid配置

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

mysql配置
```properties
#连接空闲的最大时间(s)，超出后，mysql服务端会关闭连接
wait_timeout=300
```

原因是mysql服务端主动关闭了连接，去掉wait_timeout参数(默认8h)，重启mysql和应用，连接关闭数正常了
