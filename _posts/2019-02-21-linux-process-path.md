---
layout: post
title: Linux下查看进程信息
categories: [编程, linux]
tags: [进程]
---


> 忘了`kibana`安装在什么目录了，只好根据进程端口来查找

#### 1. 查找进程ID

根据进程名称查找
```
$ ps -ef | grep xxx
```

> 通过名称查找很常用，但有多个同名进程时就不好区分了

通过端口查找

```
$ netstat -antp |grep 8080
```

#### 2. 使用pwdx查看进程工作目录

```
$ pwdx 60062
60062: /home/fool/monitor/apps/kibana-6.2.4-linux-x86_64
```

> pwdx - report current working directory of a process

#### 3. 查看/proc

```
$ ls -al /proc/${pid}

lrwxrwxrwx  1 fool fool 0 Feb 21 10:32 cwd -> /home/fool/monitor/apps/kibana-6.2.4-linux-x86_64
lrwxrwxrwx  1 fool fool 0 Feb 21 10:32 exe -> /home/fool/monitor/apps/kibana-6.2.4-linux-x86_64/node/bin/node
```

> `cwd`即进程工作目录，`exe`即进程启动脚本路径，都是软链接，指向`kibana`安装目录

#### 4. 查看进程cpu内存

```
$ ps -p ${pid} -o %cpu,%mem,rss
```