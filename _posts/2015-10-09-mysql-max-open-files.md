---
layout: post
title: "Mysql Changed limits: max_open_files: 1024 错误"
categories: [编程, mysql]
tags: []
---

> `CentOS 7 Mysql 5.6`，调整参数后重启`mysql`，日志中出现如下

```
[Warning] Buffered warning: Changed limits: max_open_files: 1024 (requested 5010
[Warning] Buffered warning: Changed limits: max_connections: 214 (requested 1000)
[Warning] Buffered warning: Changed limits: table_open_cache: 400 (requested 2000)
```

看日志是`ulimit`设置不当，查看`limits`，发现参数正确

```
ulimit -a | grep open

open files                      (-n) 65535

vi /etc/security/limits.conf

* hard nofile 65535  
* soft nofile 65535 
```

上网搜索，发现还需要修改`mysqld.service`

```
vi /usr/lib/systemd/system/mysqld.service

LimitNOFILE=65535  
```

重启`mysql`，问题解决