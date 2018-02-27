---
layout: post
title: Linux定时任务crontab
categories: [编程, linux]
tags: [crontab]
---

> `linux`中使用`crontab`实现定时任务

#### 1. 需求
每天`23:30`重启应用，每隔`3分钟`检查一次应用进程状态

#### 2. 配置crontab

编辑 `vi /etc/crontab`
```
# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed

# 每天23:30重启应用
30 23 * * * root /root/bin/restart.sh >/dev/null 2>&1 &
# 每隔3分钟检查一次应用进程状态
*/3 * * * * root /root/bin/check.sh >/dev/null 2>&1 &

```

#### 3. 重启crontab服务

重启`crontab`服务使配置生效

```
service crond restart
```