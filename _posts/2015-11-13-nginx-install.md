---
layout: post
title: CentOS下安装nginx
categories: [编程, nginx, web]
tags: []
---

#### 1.环境检查
安装gcc以及gcc++

```
yum install gcc
yum install gcc-c++
```

#### 2 安装pcre
安装pcre模块

```
yum install pcre pcre-devel
```

#### 3 安装nginx

##### 3.1 下载nginx安装包
> 在nginx官网下载最新版源码安装包

##### 3.2 解压安装
```shell
tar -zxvf nginx-1.9.9.tar.gz
cd nginx-1.9.9

./configure  --with-http_stub_status_module  --prefix=/usr/local/nginx --with-pcre=/usr/local/pcre-8.37
make
make install
```

#### 4 启动nginx
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

[root@centos01 sbin]#ps -ef | grep nginx
root      3333     1  0 11:03 ?        00:00:00 nginx: master process ./nginx
nobody    3334  3333  0 11:03 ?        00:00:00 nginx: worker process
root      6686  2785  0 15:47 pts/0    00:00:00 grep nginx
```

可以看到进程启动成功。

#### 5 关闭nginx

```
./nginx –s stop
```

#### 6 配置热加载
nginx提供命令热加载配置
```linux
nginx -s reload
```
> 该命令会用新配置重新生成worker进程，然后等待旧的worker进程处理完之前的请求后，关闭旧的worker进程。   
> 所以可以实现服务无中断重启。