---
layout: post
title: Linux下使用iptables实现ssh端口转发
categories: [编程, linux]
tags: [iptables, ssh]
---


> 有一台`Linux`主机，想把`sshd`端口从`22`改到`2200`，但又不能重启`sshd`服务，可以通过`iptables`做端口转发

#### 1.添加转发规则

添加路由规则，把所有从端口`2200`进来的`tcp`连接都转发到`22`端口
```
$ iptables -A PREROUTING -t nat -p tcp --dport 2200 -j REDIRECT --to 22
```

查看路由列表
```
$ iptables -L PREROUTING -n -t nat --line-numbers
Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    REDIRECT   tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2200 redir ports 22

$ iptables -S PREROUTING 1 -t nat
-A PREROUTING -p tcp -m tcp --dport 2200 -j REDIRECT --to-ports 22
```

#### 2. 屏蔽原22端口

```
$ iptables -A PREROUTING -t mangle -p tcp --dport 22 -j DROP
```

#### 3. 关于iptables

##### 3.1 概念

`netfilter`: 一个`Linux`防火墙框架，用于处理网络数据包
`iptables`: 用于操作`netfilter`的用户接口(客户端)

##### 3.2 rule

> `rule`是一个具体的规则，用于指明为某一类(条件)报文做何种处理

如下：表示所有来源于ip:`192.168.56.100`并且端口为`22`的`TCP`协议的`IP`报文都会被丢弃(`DROP`)
```
-s 192.168.56.100/24 -p tcp --dport 22 -j DROP
```

##### 3.3 chain

> `chain`是一个链，一条链上可以有多个规则，类似于`java-servlet`中的`filter chain`

`iptables`中`chain`一般分为如下几种：

`PERROUTING`: 前置处理链，用于执行路由规则之前
`INPUT`: 输入链，用来处理发给本机的数据包
`OUTPUT`: 输出链，用来处理本机发出的数据包
`FORWARD`: 转发链，用来处理通过本机转发的数据包
`POSTROUTING`: 后置处理链，用于执行路由规则之后

##### 3.4 table

`table`是规则的分组，一般分为以下四种：

`raw`: 决定数据包是否被状态跟踪机制处理，可用链有`PREROUTING, OUTPUT`
`mangle`: 修改数据包内容，可用链有`PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING`
`nat`: 用于网络地址转换（IP、端口），可用链有`PREROUTING, INPUT, OUTPUT, POSTROUTING`
`filter`: 过滤数据包，可用链有`INPUT, FORWARD, OUTPUT`

`netfilter`工作原理图：

![]({{site.url}}/public/images/2018-10-24-iptables-ssh-forward.png)

#### 4. 常用命令

##### 4.1 查看指定table和chain中的规则列表

```
$ iptables -L PREROUTING -n -t nat --line-numbers
```

##### 4.2 查看规则详情

```
$ iptables -S PREROUTING 1 -t nat
```

> 这里的`1`为4.1中规则在列表中的编号

##### 4.3 添加规则

```
$ iptables -[A|I] PREROUTING -t nat -p tcp --dport 2200 -j REDIRECT --to 22
```

> `-A`的添加到`chain`的最后，`I`是插入到`chain`的开始，区别是优先级不同

##### 4.4 删除规则

```
# 先查出规则的编号
$ iptables -L PREROUTING -n -t nat --line-numbers
# 再删除指定编号的规则
$ iptables -D PREROUTING 1 -t nat
```

##### 4.5 清除规则

清除所有规则，不会处理默认的规则
```
$ iptables -F
```

##### 4.6 备份和恢复

```
$ iptables-save > iptables.bak

$ iptables-restore < iptables.bak
```

#### 5. 参考

* [netfilter/iptables简介](https://segmentfault.com/a/1190000009043962)
* [iptables详解](http://www.zsythink.net/archives/tag/iptables/)
* [netfilter/iptables 简介](https://www.ibm.com/developerworks/cn/linux/network/s-netip/index.html)
