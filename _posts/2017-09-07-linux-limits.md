---
layout: post
title: Linux中的limits
categories: [编程, linux]
tags: [limits, ssh]
---

> `linux`中可以限制用户对系统资源的使用

#### 1. ulimit

用于查看当前用户的资源限制

```
$ ulimit -a

core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 127900
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 65536
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 4096
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

#### 2. limits.conf格式

```
username|@groupname type resource limit

```

##### 2.1  type

`type`可选`soft|hard|-`

* `soft`指的是当前系统生效的设置值
* `hard`表明系统中所能设定的最大值
* `-`表示同时设置`soft`和`hard`

> `soft`不能大于`hard`

##### 2.2 resource

* `core`: 限制内核文件的大小
* `date`: 最大数据大小
* `fsize`: 最大文件大小
* `memlock`: 最大锁定内存地址空间
* `nofile`: 打开文件的最大数目
* `rss`: 最大持久设置大小
* `stack`: 最大栈大小
* `cpu`: 以分钟为单位的最多`CPU`时间
* `nproc`: 进程的最大数目
* `as`: 地址空间限制
* `maxlogins`: 此用户允许登录的最大数目

#### 3. 修改limits.conf

例如修改用户可打开文件数：

```
$ vi /etc/security/limits.conf

foo    soft     nofile      65535
foo    hard     nofile      65535
```

#### 4. 如何生效

如果要重新例如生效，需要修改`sshd`配置

```
$ vi /etc/ssh/sshd_config

UsePAM yes
UseLogin yes

$ service sshd restart
```