---
layout: post
title: Linux下使用redsocks和iptables实现全局socks代理
categories: [编程, linux]
tags: [redsocks, iptables]
---

> linux下配置指定ip的访问走socks代理，以ubuntu为例

#### 1. 安装redsocks

```
$ sudo apt-get install redsocks
```

也可通过源码安装[redsocks](https://github.com/darkk/redsocks)

```
$ git clone https://github.com/darkk/redsocks.git
$ cd redsocks
$ yum install libevent-devel
$ make
```

#### 2. 配置redsocks

编辑/etc/redsocks.conf
```
redsocks {
    local_ip = 127.0.0.1;
    local_port = 12345;
    
    ip = 192.168.1.100;
    port = 1080;
    
    type = socks5;
}
```

#### 3. 重启redsocks

```
$ service redsocks restart
$ netstat -an|grep 12345
tcp        0      0 127.0.0.1:12345         0.0.0.0:*               LISTEN
```

#### 4. 配置iptables

配置iptables，重定向所有172.16.200.0/24网段的出口访问到redsocks

```
$ sudo iptables -t nat -A OUTPUT -d 172.16.200.0/24 -p tcp -j REDIRECT --to-ports 12345
```

#### 5. 参考

* [redsocks](https://github.com/darkk/redsocks)
* [redsocks 配合iptables设置全局sockts5代理](https://www.cnblogs.com/cmsd/p/4363631.html)


