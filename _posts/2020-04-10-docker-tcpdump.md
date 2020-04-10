---
layout: post
title: 使用tcpdump对docker容器进程抓包
categories: [编程, linux, docker]
tags: [nsenter, tcpdump]
---


> 容器内已有`tcpdump`命令的情况不在本文讨论的范围

#### 1. 查看容器进程
{% raw %}
```
$ docker ps|grep nginx
02565392211b        nginx:1.16.1
$ docker inspect --format "{{.State.pid}}" 02565392211b
19584
```
{% endraw %}
#### 2. 进入进程的网络命名空间

```
$ nsenter -n -t 19584
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
6180: eth0@if6181: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:1e:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.30.0.2/16 brd 172.30.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

> 可以通过`nsenter -n -t ${pid}`命令进入指定进程的网络命名空间


#### 3. 使用tcpdmp抓包

略...