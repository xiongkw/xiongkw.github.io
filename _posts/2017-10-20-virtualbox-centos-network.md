---
layout: post
title: VirtualBox中配置CentOS虚拟机网络
categories: [编程, 虚拟机]
tags: [网络, virtualbox]
---


> 相比`vmware`，`VirtualBox`是一个比较轻量级的虚拟机软件，同时由于其不收费，省掉了一堆寻找`license`的麻烦

#### 1. VirtualBox的网络模式

* `NAT`: 网络地址转换，虚拟机可访问外网，但虚拟机之间不可访问，虚拟机和主机之间不可访问
* `Bridged Adapter`: 网桥模式，需要为每台虚拟机分配独立的外网`ip`，虚拟机和主机可相互访问，虚拟机之间也可相互访问
* `Internal`: 内网模式，仅虚拟机之间可相互访问
* `Host-only Adapter`: 仅主机网络，主机和虚拟机之间可相互访问，虚拟机之间也可相互访问，但虚拟机不可访问外网

#### 2. 启用Host-only

在虚拟机设置界面，启用`网卡1`，并选择`Host-only`

启动虚拟机(`CentOS`)后进入，查看网卡`enp0s3`(`ip addr`)，发现此时是没有`ip`的

配置网卡参数：
```
$ vi /etc/sysconfig/network-scripts/ifcfg-enp0s3

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s3
UUID=e527a9cf-269c-4247-90a6-6309bd93e30b
DEVICE=enp0s3
ONBOOT=yes
```

> 这里以`CentOS7-Minimal`为例

重启网络服务，再次查看网卡`enp0s3`(`ip addr`)，现在有`ip`了
```
service network restart
```

#### 3. 启用NAT

`Host-only`网络下，虚拟机默认是不可访问外网的，可通过`网卡共享`或者`桥接`实现，这里使用`NAT`实现

在虚拟机设置界面，启用`网卡2`，并选择`NAT`

进入虚拟机，发现多了一个网卡`enp0s8`，并且可以访问外网
