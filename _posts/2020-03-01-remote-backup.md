---
layout: post
title: 使用sshpass和crond实现异地数据备份
categories: [编程, linux]
tags: [sshpass]
---


> 一种简单的异地数据备份方法

#### 1. 安装sshpass

`sshpass`可以在命令行使用密码直接执行`ssh`和`scp`命令

```
$ yum install sshpass -y
```

> 也可以配置`ssh`免密登录

#### 2. 编写备份脚本

本文以`mysql`为例

```
$ vi backup.sh

#!/bin/bash

DIR=$(cd $(dirname $0); pwd)
mysqldump -uroot -proot -h 127.0.0.1 -P3306 --default-character-set=utf8 demo > demo.sql

DATE=`date +%Y-%m-%d`
tar zcvf /data/backup/mysql/demo-${DATE}.tar.gz demo.sql --remove-files

sshpass -p '123456' scp -P22 /data/backup/mysql/demo-${DATE}.tar.gz root@192.168.1.101:/data/backup/mysql/
```

#### 3. 定时器

每天凌晨4点执行

```
$ echo "0  4  *  *  *   root   /data/backup/mysql/backup.sh > /data/backup/mysql/backup.log" >> /etc/crontab
$ systectl restart crond
```