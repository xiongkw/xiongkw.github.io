---
layout: post
title: CentOS定时任务
categories: [编程, linux]
tags: [crond]
---

> `CentOS`中使用`crond`服务管理定时任务

#### 1. 几个重要文件

* `/etc/anacrontab`: 系统级别任务
* `/etc/cron.d/`: 系统级别任务
* `/var/spool/cron/`: 用户级别任务，可在此目录下查看和管理所有用户的定时任务

#### 2. crond的启停

```
$ systemctl start crond
$ systemctl stop crond
$ systemctl restart crond
```

#### 3. crontab命令

用户可以使用`crontab`命令管理自己的定时任务

```
$ crontab -e
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * command to be executed
  0  *  *  *  * /home/mytest.sh >/dev/null 2>&1 &
```

编辑完成后可在`/var/spool/cron`目录下查看任务

```
$ cat /var/spool/cron/foo
  0  *  *  *  * /home/mytest.sh >/dev/null 2>&1 &
```

#### 4. 配置crontab

可直接修改`/etc/crontab`添加系统级别的定时任务
```
$ vi /etc/crontab

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
  0  *  *  *  * foo /home/mytest.sh >/dev/null 2>&1 &
```

> 注意`/etc/crontab`表达式中多了`user-name`

#### 5. 重启crontab服务

系统会自动加载新添加的定时任务，也可以重启`crontab`服务使配置立即生效

```
$ systemctl restart crond
```