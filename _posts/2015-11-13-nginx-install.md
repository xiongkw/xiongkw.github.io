---
layout: post
title: Nginx安装指南
categories: [编程, web, nginx]
tags: [nginx, web]
---

## 1.环境检查
判断系统是否已经安装gcc环境

```shell
rpm -qa|grep gcc
```

```
[root@centos01 sbin]#rpm -qa|grep gcc
libgcc-4.4.7-17.el6.x86_64
```

没有gcc以及gcc++，需要安装:

```
yum install gcc
yum install gcc-c++
```

安装好之后，再次验证下：

```
[root@centos01 sbin]#rpm -qa|grep gcc
gcc-c++-4.4.7-17.el6.x86_64
libgcc-4.4.7-17.el6.x86_64
gcc-4.4.7-17.el6.x86_64
```

## 2 安装pcre
nginx安装，需要依赖于pcre模块。

```
yum install pcre pcre-devel
```

## 3 安装nginx

### 3.1 下载nginx安装包

<a href="assets/nginx-1.9.9.tar.gz" target="_blank">点此下载nginx-1.9.9.tar.gz</a>。

### 3.2 解压安装
解压
```shell
tar -zxvf nginx-1.9.9.tar.gz
cd nginx-1.9.9
```
安装
```
./configure  --with-http_stub_status_module  --prefix=/usr/local/nginx --with-pcre=/usr/local/pcre-8.37
make
make install
```

## 4 启动nginx

进入sbin目录：

```
cd /usr/local/nginx/sbin
```

查看nginx版本，以及configure参数

```
./nginx –V
```

启动nginx

```
./nginx
```

查看nginx进程：

```
ps -ef | grep nginx
```

```
[root@centos01 sbin]#ps -ef | grep nginx
root      3333     1  0 11:03 ?        00:00:00 nginx: master process ./nginx
nobody    3334  3333  0 11:03 ?        00:00:00 nginx: worker process
root      6686  2785  0 15:47 pts/0    00:00:00 grep nginx
```

可以看到进程启动成功。

## 5 关闭nginx

```
./nginx –s stop
```

## 6 配置热加载
nginx提供命令热加载配置
```linux
nginx -s reload
```
> 该命令会用新配置重新生成worker进程，然后等待旧的worker进程处理完之前的请求后，关闭旧的worker进程。   
> 所以可以实现服务无中断重启。