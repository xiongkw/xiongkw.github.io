---
layout: post
title: CentOS下使用Squid搭建HTTP代理服务器
categories: [编程, linux]
tags: [squid, proxy]
---


> 一般公司都会有一组只能上内网的服务器，但也会提供一台能上公网的主机，我们可以搭建`HTTP`代理服务器，使内网主机也能访问外网

#### 1. Squid安装

yum安装
```
$ yum install squid -y

```

配置
```
$ vi /etc/squid/squid.conf

# 代理端口
http_port 8000

# acl 名单
acl client src 192.168.1.100

# allow
http_access allow client

# deny
http_access deny all

```

启动
```
service squid start
```

#### 2. 客户端配置

##### 2.1 HTTP代理

修改环境变量
```
export http_proxy=http://192.168.1.100:8000
export https_proxy=http://192.168.1.100:8000
```

##### 2.2 yum代理

修改`/etc/yum.conf`
```
proxy=http://192.168.1.100:8000
```

#### 2.3 wget代理

```
http_proxy=http://192.168.1.100:8000
https_proxy=http://192.168.1.100:8000
ftp_proxy=http://192.168.1.100:8000
```

#### 2.4 git代理
```
git config --global https.proxy http://192.168.1.100:8000
```

#### 3. 参考

[Squid Documentation](http://www.squid-cache.org/Doc/)