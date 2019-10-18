---
layout: post
title: 通过rsync和inotify实现双向数据实时同步
categories: [编程, linux]
tags: [rsync, inotify]
---

> 部署双机高可用时，需要同步两个实例上的数据文件

#### 1. 环境准备

```
192.168.1.101
192.168.1.102
```

#### 2. 部署rsync服务端

以`192.168.1.101`为例，在`192.168.1.101`上部署`rsync`服务

##### 2.1 安装rsync

`CentOS7`自带，略...

##### 2.2 配置rsync.conf

编辑`/etc/rsync.conf`

```
uid = root
gid = root
use chroot = yes
max connections = 4
pid file = /var/run/rsyncd.pid
exclude = lost+found/
transfer logging = yes
timeout = 900
ignore nonreadable = yes
dont compress   = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2

[share]
        path = /data/share
        comment = share
        auth users = rsync:rw
        secrets file= /app/dsync/rsyncd.secrets
```

> 配置数据同步的目录`/data/share`

##### 2.3 编写密钥文件

编辑`/app/dsync/rsyncd.secrets`

```
$ vi rsyncd.secrets
rsync:rsync
$ chmod 600 rsyncd.secrets
```

> 保存用户账号，客户端即可用该账号同步文件，格式为`username:password`，每行一个账号，权限必须为`600`

##### 2.4 启动rsyncd服务

编写系统服务(`/usr/lib/systemd/system/rsyncd.service`)：

```
[Unit]
Description=fast remote file copy program daemon
ConditionPathExists=/etc/rsyncd.conf

[Service]
EnvironmentFile=/etc/sysconfig/rsyncd
ExecStart=/usr/bin/rsync --daemon --no-detach "$OPTIONS"

[Install]
WantedBy=multi-user.target
```

启动：

```
$ systemctl daemon-reload
$ systemctl start rsyncd
```

##### 2.5 测试

以`192.168.1.101`主机为服务器，在`192.168.1.102`上执行

1. 同步服务端文件到客户端

```
$ rsync -avz rsync@192.168.1.101::share /data/share --password-file=/app/dsync/rsyncd.passwd
```

`/app/dsync/rsyncd.passwd`用于指定`rsync`用户的密码，内容

```

rsync
```

2. 同步客户端文件到服务端

```
$ rsync -avz /data/share rsync@192.168.1.101::share --password-file=/app/dsync/rsyncd.passwd
```

##### 2.6 部署另一台主机

在主机`192.168.1.102`上部署`rsync`服务，略...

> 到现在为止，两台主机都运行了`rsync`服务，并提供了数据同步的能力，但是只能靠人工触发，无法做到自动化

#### 3. 部署监听服务

`inotify`可用于监听文件变化，从而触发`rsync`实时同步，以`192.168.1.101`为例

##### 3.1 安装inotify-tools

```
$ yum install -y epel-release && yum update
$ yum install inotify-tools
$ inotifywait --help
```

##### 3.2 编写监听脚本

编辑`/app/dsync/dsync.sh`

```
#!/bin/bash

USER=rsync
SRC=/data/share/
DST=$USER@192.168.1.102::share

/usr/bin/inotifywait -mrq -e modify,delete,create,attrib $SRC | while read file
do
        rsync -vzrtopg --delete $SRC $DST --password-file=/app/dsync/rsyncd.passwd
        echo "${file} was rsynced" >> /tmp/rsync.log 2>&1
done
```

> 监听`modify,delete,create,attrib`事件   
> 实时监听并同步数据到客户机，所以库里的`SRC`为本机，`DST`为目的主机   
> 注意`SRC`必须以`/`结尾，否则同步到`DST`会多出一级`share`目录

添加执行权限

```
$ chmod 755 dsync.sh
```

##### 3.3 运行服务

编写系统服务(`/usr/lib/systemd/system/dsyncd.service`)

```
[Unit]
Description=Dsyncd service
After=network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/app/dsync/dsync.sh
Restart=on-failure
TimeoutSec=300

[Install]
WantedBy=multi-user.target

```

启动：

```
$ systemctl daemon-reload
$ systemctl start dsyncd
```

##### 3.4 测试

创建`/data/share/test`文件

```
echo "haha" > /data/share/test
```

在另一台主机(`192.168.1.102`)上查看，发现`/data/share/test`已经同步过来了

##### 3.5 部署另一台主机

在主机`192.168.1.102`上部署监听服务，略...

> 到现在为止，两台主机都运行了`dsync`服务，并提供了监听和同步数据的能力，实现了自动化的实时双向数据同步

#### 5. 参考

* [rsyncd.conf](https://download.samba.org/pub/rsync/rsyncd.conf.html)
* [inotify-tools](https://github.com/rvoicilas/inotify-tools/wiki)

