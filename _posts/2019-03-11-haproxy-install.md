---
layout: post
title: HAProxy试用
categories: [编程, linux, haproxy]
tags: []
---


> 听说`HAProxy`的反向代理性能很好，所以想了解下

#### 1. 下载安装

官网`proxy.org`被墙了，上`github`下载源码，[haproxy](https://github.com/haproxy/haproxy/releases)

解压安装

```
$ tar zxvf haproxy-1.9.0
$ uname -r
3.10.0-327.el7.x86_64
$ cd haproxy-1.9.0
$ make TARGET=linux2628 PREFIX=/usr/local/haproxy-1.9.0
$ make install PREFIX=/usr/local/haproxy-1.9.0
$ cd /usr/local/haproxy-1.9.0/sbin
$ ./haproxy -v
HA-Proxy version 1.9.0 2018/12/19 - https://haproxy.org/
```

#### 2. 运行

编写配置文件：
```
global
        maxconn         3000000
        nbproc 8
        #stats socket    /var/run/haproxy.stat mode 600 level admin
        stats bind-process
        log             127.0.0.1 local0
        chroot          /var/empty
        daemon

# http代理
frontend fron_http
        bind            192.168.1.23:8804 name clear
        mode            http
        log             global
        option          httplog
        option          dontlognull
        monitor-uri     /monitoruri
        maxconn         300000
        timeout client  30s
        default_backend back_http

backend back_http
        mode            http
        balance         roundrobin
        retries         2
        option redispatch
        timeout connect 5s
        timeout server  30s
        timeout queue   30s
        cookie          DYNSRV insert indirect nocache
        fullconn        300000 # the servers will be used at full load above this number of connections
        server          dynsrv1 192.168.1.23:8802 minconn 50 maxconn 300000 cookie s1 check inter 1000

# tcp 代理
frontend fron_tcp
        bind            192.168.1.23:8805 name clear
        mode            tcp
        log             global
        option          dontlognull
        maxconn         300000
        timeout client  30s
        default_backend back_tcp

backend back_tcp
        mode            tcp
        balance         roundrobin
        retries         2
        option redispatch
        timeout connect 5s
        timeout server  30s
        timeout queue   30s
        fullconn        300000 # the servers will be used at full load above this number of connections
        server          dynsrv1 192.168.1.61:8802 minconn 50 maxconn 300000
```

启动

```
$ sbin/haproxy -f conf/sample.cfg -p pid
```

#### 3. haproxy的配置重载

```
$ sbin/haproxy -f conf/sample.cfg -sf $(cat pid) -p pid
```

> 通过`-sf`选项，指定旧进程`ID`，新的进程启动完成后`kill`旧进程

#### 4. 参考

* [HAProxy](https://cbonte.github.io/haproxy-dconv/1.9/intro.html)

* [HAProxy Enterprise Documentation](https://www.haproxy.com/documentation/hapee/)

> 开源版被墙，只能看商业版