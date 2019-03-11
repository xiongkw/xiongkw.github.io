---
layout: post
title: HAProxy试用
categories: [编程, linux, haproxy]
tags: []
---


> 听说`HAProxy`的反向代理性能要优于`nginx`，所以想了解下

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
        maxconn         10000
        stats socket    /var/run/haproxy.stat mode 600 level admin
        log             127.0.0.1 local0
        chroot          /var/empty
        daemon

# The public 'www' address in the DMZ
frontend public
        bind            192.168.1.10:8080 name clear
        mode            http
        log             global
        option          httplog
        option          dontlognull
        monitor-uri     /monitoruri
        maxconn         8000
        timeout client  30s

        stats uri       /admin/stats
        use_backend     static if { hdr_beg(host) -i img }
        use_backend     static if { path_beg /img /css   }
        default_backend dynamic

# The static backend backend for 'Host: img', /img and /css.
backend static
        mode            http
        balance         roundrobin
        option prefer-last-server
        retries         2
        option redispatch
        timeout connect 5s
        timeout server  5s
        server          statsrv1 192.168.1.11:8080 check inter 1000

# the application servers go here
backend dynamic
        mode            http
        balance         roundrobin
        retries         2
        option redispatch
        timeout connect 5s
        timeout server  30s
        timeout queue   30s
        cookie          DYNSRV insert indirect nocache
        fullconn        4000 # the servers will be used at full load above this number of connections
        server          dynsrv1 192.168.1.12:8080 minconn 50 maxconn 500 cookie s1 check inter 1000
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

* [HAProxy](http://www.haproxy.org/)

* [HAProxy Enterprise Documentation](https://www.haproxy.com/documentation/hapee/)

> 开源版被墙，只能看商业版