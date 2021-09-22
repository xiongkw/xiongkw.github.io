---
layout: post
title: 使用frp做端口映射
categories: [编程, linux]
tags: [frp]
---

> 有一个内网服务，需要提供公网访问。通常可以使用ssh隧道实现，不过有时候公网主机并不允许提供ssh访问，这时就可以使用frp等软件实现

#### 1.名词解释

* 端口转发：把A主机的端口PA转发到B主机的端口PB，使得访问PA就能访问PB
* 端口映射：把内网主机B的端口PB映射到公网主机的端口PA，使得内网服务可以提供公网访问。映射其实也是一种转发，只不过映射强调的是公网转发到内网
* 反向代理：把A主机的端口PA代理到B主机端口PB，本质也是一种转发，只不过反向代理的重点是隐藏被代理服务

#### 2. 安装frp
下载[安装包](https://github.com/fatedier/frp)

##### 2.1 服务端(公网主机)

服务端的安装文件是frps*

```
$ tar zxvf frp_0.37.1_linux_amd64.tar.gz
$ cp frp_0.37.1_linux_amd64/frps /usr/bin
$ mkdir /etc/frp && cp frp_0.37.1_linux_amd64/frps.ini /etc/frp/frps.service
$ cp frp_0.37.1_linux_amd64/systemd/frps.service /usr/lib/systemd/system
```

##### 2.2 客户端(内网主机)

客户端的安装文件是frpc*

```
$ tar zxvf frp_0.37.1_linux_amd64.tar.gz
$ cp frp_0.37.1_linux_amd64/frpc /usr/bin
$ mkdir /etc/frp && cp frp_0.37.1_linux_amd64/frpc.ini /etc/frp/frps.service
$ cp frp_0.37.1_linux_amd64/systemd/frpc.service /usr/lib/systemd/system
```

#### 3 测试

把内网主机的8080端口映射到公网主机的8081端口

##### 3.1 配置服务端(公网主机)

修改/etc/frp/frps.ini：

```
[common]
bind_port=7000
token=123456
```

重启frps服务

```
$ systemctl restart frps
```

##### 3.2 配置客户端(内网主机)

修改/etc/frp/frpc.ini：

```
[common]
server_addr = 120.25.x.x
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 8080
remote_port = 8081
```

重启frpc服务

```
$ systemctl restart frpc
```

##### 3.3 访问公网端口

略

#### 4. 参考

* [rfp](https://github.com/fatedier/frp)