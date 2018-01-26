---
layout: post
title: Centos中使用nginx代理yum镜像
categories: [编程, linux, nginx]
tags: [centos, yum]
---

> 内网主机`Centos`需要安装软件，但无法访问公网，由于其依赖项太多，也不太可能每个都下载了再安装，好在个别主机可以访问外网，考虑架设一个代理服务器

#### 通过nginx代理

在能访问公网的主机上安装nginx

nginx.conf
```
    server {
        listen       8000;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
            proxy_pass http://mirrors.aliyun.com;
        }
     }
```

> 监听8000端口，这里代理到`mirrors.aliyun.com`

#### 配置yum源

```
vi /etc/yum.repo.d/local.repo

name=CentOS-$releasever - Base - 192.168.1.100:8000
failovermethod=priority
baseurl=http://192.168.1.100:8000/centos/$releasever/os/$basearch/
        http://192.168.1.100:8000/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
gpgcheck=1
gpgkey=http://192.168.1.100:8000/centos/RPM-GPG-KEY-CentOS-7

#released updates 
[updates]
name=CentOS-$releasever - Updates - 192.168.1.100:8000
failovermethod=priority
baseurl=http://192.168.1.100:8000/centos/$releasever/updates/$basearch/
        http://192.168.1.100:8000/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
gpgcheck=1
gpgkey=http://192.168.1.100:8000/centos/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - 192.168.1.100:8000
failovermethod=priority
baseurl=http://192.168.1.100:8000/centos/$releasever/extras/$basearch/
        http://192.168.1.100:8000/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
gpgcheck=1
gpgkey=http://192.168.1.100:8000/centos/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - 192.168.1.100:8000
failovermethod=priority
baseurl=http://192.168.1.100:8000/centos/$releasever/centosplus/$basearch/
        http://192.168.1.100:8000/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=http://192.168.1.100:8000/centos/RPM-GPG-KEY-CentOS-7

#contrib - packages by Centos Users
[contrib]
name=CentOS-$releasever - Contrib - 192.168.1.100:8000
failovermethod=priority
baseurl=http://192.168.1.100:8000/centos/$releasever/contrib/$basearch/
        http://192.168.1.100:8000/centos/$releasever/contrib/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=contrib
gpgcheck=1
enabled=0
gpgkey=http://192.168.1.100:8000/centos/RPM-GPG-KEY-CentOS-7
```
