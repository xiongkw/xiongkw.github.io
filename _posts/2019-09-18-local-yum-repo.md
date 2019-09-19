---
layout: post
title: yum本地源搭建
categories: [编程, linux]
tags: [yum]
---

> 有一堆机器，无法连外网，所以需要搭建本地的yum源服务器，以`CentOS7`为例

#### 1. yum源制作

##### 1.1 centos基础rpm包

使用`CentOS iso`中自带的基础`rpm`包

```
$ mount /dev/cdrom /mnt/cdrom
$ ll /mnt/cdrom
drwxrwxr-x. 2 root root 69632 Sep  5  2017 Packages
drwxr-xr-x. 2 root root  4096 Sep  5  2017 repodata
```

##### 1.2 其它rpm包

例如`epel`中的包，需要自己下载并制作成`yum`源

1. 下载`rpm`包及其依赖

```
$ yum install httpd --downloadonly --downloaddir=~/extras/Packages
$ ll Packages
-rw-r--r--. 1 root root  105968 Aug 22 17:20 apr-1.4.8-5.el7.x86_64.rpm
-rw-r--r--. 1 root root   94132 Jul  3  2014 apr-util-1.5.2-6.el7.x86_64.rpm
-rw-r--r--. 1 root root 2844388 Aug 22 17:25 httpd-2.4.6-90.el7.centos.x86_64.rpm
-rw-r--r--. 1 root root   92944 Aug 22 17:25 httpd-tools-2.4.6-90.el7.centos.x86_64.rpm
-rw-r--r--. 1 root root   31264 Jul  3  2014 mailcap-2.1.41-2.el7.noarch.rpm
```

2. 安装`createrepo`

```
$ yum install createrepo -y
```

3. 创建源

```
$ createrepo ~/extras
$ ll 
drwxr-xr-x. 2 root root  212 Sep 18 04:27 Packages
drwxr-xr-x. 2 root root 4096 Sep 18 04:31 repodata
```

4. 更新源

下载了其它`rpm`包，只需要更新源即可

```
$ createrepo --update ~/extras
```

#### 2. 服务搭建

需要搭建`http`服务，以`httpd`为例

##### 2.1 安装httpd

```
$ yum install httpd -y
```

##### 2.2 部署yum源

直接`copy`到`httpd`默认目录

```
$ mkdir -p /var/www/html/centos/7/base
$ cp /mnt/cdrom/Packages /var/www/html/centos/7/base -r
$ cp /mnt/cdrom/repodata /var/www/html/centos/7/base -r
$ cp ~/extras /var/www/html/centos/7/ -r
```

##### 2.3 启动httpd

需要关闭防火墙和`selinux`

```
$ systemctl stop firewalld
$ setenforce 0
$ systemctl start httpd
```

#### 3. 本地源的使用

1. 备份本机的`yum`源配置

```
$ mkdir /etc/yum.repos.d/bak
$ mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak
```

2. 增加本地源配置

```
$ vi /etc/yum.repos.d/local.repo
```

内容

```
[local-base]
name=local-base
baseurl=http://192.168.1.101/centos/7/base/
gpgcheck=0

[local-extras]
name=local-extras
baseurl=http://192.168.1.101/centos/7/extras
gpgcheck=0
```

3. 测试

```
$ yum clean all
$ yum list net-tools
Available Packages
net-tools.x86_64             2.0-0.22.20131004git.el7       local-base
$ yum list httpd
Available Packages
httpd.x86_64                 2.4.6-90.el7.centos            local-extras
```

#### 4. 附. 离线安装httpd

既然无法连接外网，那么如何安装`httpd`呢?

先在可以连外网的主机上下载`httpd`及其依赖包

```
$ yum install httpd --downloadonly --downloaddir=~/httpd
```

`copy`下载到的安装包到目标主机上再安装

```
$ yum install httpd/*
```