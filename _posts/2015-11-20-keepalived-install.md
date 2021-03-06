---
layout: post
title: Keepalived安装指南
categories: [编程, linux]
tags: [lvs, 负载均衡, keepalived]
---


#### 1. 系统要求
`lvs` (`linux`内核2.6版本以上已经集成了`lvs`，不需要再额外安装)

#### 2. 安装基础环境
* 安装`gcc`

```
yum install gcc gcc-c++
```

* 安装基础依赖包

```
yum install make openssl openssl-devel kernel-devel popt-dev
```

#### 3. 安装ipvsadmin

* 检查`kernel`是否有`ipvs`模块

运行命令，检查是否有输出
```
modprobe -c |grep ip_vs
```

* 安装`ipvsadmin`

```
yum install -y ipvsadm
```

* 验证安装是否成功

```
ipvsadm –help
```

#### 4. 安装keepalived
* 下载安装

下载[keepalived-1.2.13](http://www.keepalived.org/software/keepalived-1.2.13.tar.gz)
```
1)  tar -zxvf keepalived-1.2.13.tar.gz
2)  cd keepalived-1.2.13
3)  ./configure --prefix=/usr/local
4)  make
5)  make install
```

* 配置

配置`keepalived`为系统服务：
```
1)  cp /usr/local/sbin/keepalived /usr/bin/
2)  cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
3)  cp /usr/local/etc/rc.d/init.d/keepalived /etc/init.d/
4)  mkdir /etc/keepalived
5)  cp /usr/local/etc/keepalived/keepalived.conf /etc/keepalived
```

> 配置完成后可通过`service keepalived (start|stop|restart)` 对`keepalived`程序进行管理