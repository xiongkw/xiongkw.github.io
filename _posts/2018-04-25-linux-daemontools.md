---
layout: post
title: Linux下使用daemontools管理进程
categories: [编程, linux]
tags: [daemontools, storm]
---


> `Linux`下自动启动进程的方法有很多，这里使用`daemontools`

#### 1. 简介

> daemontools is a collection of tools for managing UNIX services.   
> supervise monitors a service. It starts the service and restarts the service if it dies. Setting up a new service is easy: all supervise needs is a directory with a run script that runs the service.


#### 2. 安装

##### 2.1 下载
到`Daemontools`官网下载安装包

[下载](http://cr.yp.to/daemontools/install.html)

##### 2.2 安装

解压安装
```
tar zxvf daemontools-0.76.tar.gz

cd admin/daemontools-0.76

package/install
```

> `CentOS7`下会出现错误: `/usr/bin/ld: errno: TLS defini  tion in /lib/libc.so.6 section .tbss mismatches non-TLS reference in envdir.o`   
> 解决方法: 修改`admin/daemontools-0.76/src/error.h`中的`extern int errno;`为`#include <errno.h>`

#### 3. 启动

安装好`daemontools`后会创建两个目录
```
/command
    envdir //在目录里运行利用环境变量提供的程序
    envuidgid //完成setuidgid程序的功能，但是设置环境变量$UID 和 $GID 帐号提供的UID和GI
    fghack //防止进程把自己放到后台运行
    multilog //是登陆程序。它从daemon获得输出，添加到log文档里
    pgrphack /在单独的进程组里开启一个进程
    readproctitle //显示ps输出的log文件
    setlock //锁住可执行程序的文件
    setuidgid //在指定的帐号下运行一个特定的程序
    softlimit //对指定程序允许资源限制
    supervise //运行 svscan给它的运行脚本，监听脚本开始的进程，当进程死亡的时候，使之重新运行
    svc //发送信号到在supervise下运行的进程
    svok //检查目录中运行的 supervise 
    svscan //检查服务目录，为每一个找到的脚本开始管理进程
    svscanboot //是一个普通的脚本， 用来运行svscan，把输出定向到readproctitle
    svstat //显示supervise监听到的进程的状态
    tai64n //是一个timestamp生成程序
    tai64nlocal //改变 tai64n 输出文件到可读的格式
    
/service
```

启动`svscanboot`
```
/command/svscanboot &
```
> `svscanboot`负责启动`svscan`服务，`svscan`管理`supervise`进程。而具体的客户进程，是通过`supervise`进程来统一管理的

#### 4. 管理进程
这里以`storm supervisor`进程为例

##### 4.1 编写`supervisor run`
```
$ vi supervisor/run

#!/bin/bash

exec ../storm supervisor > /dev/null 2>&1
```

##### 4.2 创建软链接
```
ln -s /apps/apache-storm-1.2.1/bin/supervisor/ /service/
```

##### 4.3 查看进程

可以看到`storm supervisor`进程已经自动启动了
```
ps aux|grep storm
```

> `kill`掉进程后，发现进程也会自动启动

##### 4.4 kill进程

使用`daemontools`管理进程后，发现`kill`掉的进程马上又会自动重启，如果不希望进程重启，可以暂时`kill`掉`daemontools`进程

```
svscanboot
svscan
supervise
```

##### 4.5 关于环境变量

常见的问题是运行时找不到指定命令，原因是`svscanboot`会把`PATH`设置为一个默认值

> [The svscanboot program](http://cr.yp.to/daemontools/svscanboot.html)

`daemontools`提供`envdir`方式来解决，参考[Setting Environment Variables](http://troubleshooters.com/linux/djbdns/daemontools_intro.htm#setting_environment_variables)

这里提供一个简单的解决办法：编辑`svscanboot`修改`PATH`

```
$ vi daemontools/command/svscanboot

PATH=$PATH:/command:/usr/local/bin:/usr/local/sbin:/bin:/sbin:/usr/bin:/usr/sbin:/usr/X11R6/bin
```

#### 5. 参考文档

* [daemontools](http://cr.yp.to/daemontools.html)

