---
layout: post
title: lvs持久连接的问题
categories: [编程, linux]
tags: [lvs, keepalived, tcp]
---

#### 1. 现象

`keepalived`主备切换后`real server`的`tcp`连接数翻倍

#### 2. 测试

测试环境
```
keepalived
  MASTER: 21
  BACKUP: 23
  VIP: 245
Real Server: 61(mysql)
Client: 22(mysql)

```

##### 2.1 `keepalived`主备切换
- `mysql`客户端正常连接`vip`

- 查看`RS TCP`
```
tcp6       0      0 10.142.90.245:8806      10.142.90.22:22178      ESTABLISHED
```

- 查看客户机`TCP`
```
tcp        0      0 10.142.90.22:22178      10.142.90.245:8806      ESTABLISHED
```

- 查看`MASTER`上`ipvs`持久连接情况

```
ipvsadm -lnc

IPVS connection entries
pro expire state       source             virtual            destination
TCP 01:53  ESTABLISHED 10.142.90.22:22178 10.142.90.245:8806 10.142.90.61:8806
TCP 00:43  NONE        10.142.90.22:0     10.142.90.245:8806 10.142.90.61:8806
```

- 关闭`MASTER keepalived`进程使主备切换

```
service keepalived stop
```

- 操作`mysql`客户端，发现有一次断线重连

```
mysql> show databases;
ERROR 2013 (HY000): Lost connection to MySQL server during query
mysql> show databases;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    14370
Current database: *** NONE ***
```

- 查看`RS TCP`

```
tcp6       0      0 10.142.90.245:8806      10.142.90.22:22178      ESTABLISHED
tcp6       0      0 10.142.90.245:8806      10.142.90.22:24781      ESTABLISHED
```

- 查看客户机`TCP`

```
tcp        0      0 10.142.90.22:24781      10.142.90.245:8806      ESTABLISHED
```

- 查看现`MASTER`上`ipvs`持久连接

```
ipvsadm -lnc

IPVS connection entries
pro expire state       source             virtual            destination
TCP 01:04  ESTABLISHED 10.142.90.22:24781 10.142.90.245:8806 10.142.90.61:8806
TCP 01:12  NONE        10.142.90.22:0     10.142.90.245:8806 10.142.90.61:8806
```

> 可以发现客户端端口改变，`mysql`也提示断线重连，而`RS`上也出现两个连接，即先前的连接没有关闭

##### 2.2 lvs persistent
`LVS持久连接`：当使用`LVS`持久连接时，`director`使用一个连接跟踪（持久连接模板）使所有来自同一客户端的连接被标记为相同的`realserver`；而当客户端向集群服务器请求连接时，`director`会查看持久连接模板，是否`realserver`已经被标记为这种类型的连接，若没有，`director`重新为每次连接创建一个正常连接的跟踪记录表（持久连接模板）

持久连接类型：

- `PCC（persistent client connections）`：持久的客户端连接(零端口的持久连接)，使来自于同一个客户端对所有服务的请求，始终定向至此前选定的`realserver`
- `PPC（persistent port connections）`：持久的端口连接，使来自于同一个客户端对同一个服务的请求，始终定向至此前选定的`realserver`
- `Persistent Netfilter Marked Packet persistence`：基于防火墙标记的持久性连接，使来自于同一客户端对指定服务的请求，始终定向至此算定的`realserver`基于指定的端口，它可以将两个毫不相干的端口定义为一个集群服务，例如：合并`http telnet`为同一个集群服务

持久连接参数：

* `ipvsadm --set tcp tcpfin udp`
* `ipvsadm --persistent timeout`

##### 2.3 关于持久时间的测试

修改持久时间，重新测试

```
ipvsadm --set 120 60 300

ipvsadm -E -t 10.142.90.245:8806 -p 60
```

> 发现在持久时间过了以后，`ipvs`会清空持久连接模板，这时候`mysql`客户端也会断线重连，`2.1`的问题重现，说明其原因是由`lvs`持久连接引起

#### 3. TCP抓包分析

在持久连接失效后再次访问`mysql`客户端，并分别在`lvs`主机和`rs`主机上抓包

##### 3.1 在lvs主机抓包

```
tcpdump host 10.142.90.245 -w a.cap
```

![]({{site.url}}/public/images/2017-12-27-lvs-persistent-timeout-1.png)

> 通过`mac`地址可看到`lvs`主机收到第一个包后，响应了一个`RST`包，表示旧的连接已经失效了，需要建立新的连接   
> 这里有一些`TCP Out-Of-Order` `TCP Dup ACK` `TCP Retransmission`的错误，仔细比较发现其`src mac`和`dst mac`是不同的，原因是`DR`模式修改了包中的`mac`地址，也就是说两个`重复的`包中有一个其实是发往`rs`的，只不过这里被`WireShark`认为是`重复的`了

##### 3.2 在real server主机抓包

```
tcpdump host 10.142.90.245 -w b.cap
```

![]({{site.url}}/public/images/2017-12-27-lvs-persistent-timeout-2.png)

> 通过与lvs主机`tcp dump`比较，可以看到`lvs`主机上第一个包并没有转发到`rs`主机

##### 3.3 总结

`lvs`持久连接失效后，`lvs`收到旧的连接发送`TCP`包后，会响应一个`RST`包，表示旧的连接已经失效，需要建立新的连接

#### 4. 解决方法

> 查找`lvs`文档，没有发现什么参数能够设置让`lvs`主动通知`rs`去关闭失效的持久连接。理论上，如果持久连接失效后，`lvs`通知`rs`关闭连接也会存在问题，因为此时`client`虽然没有发送数据，但有可能正在接收数据。

修改rs主机`net.ipv4.tcp_keepalive_time`，让连接尽快回收

```
net.ipv4.tcp_keepalive_time=120 
```

* `tcp_keepalive_time` // 距离上次传送数据多少时间未收到判断为开始检测
* `tcp_keepalive_intvl` // 检测开始每多少时间发送心跳包
* `tcp_keepalive_probes` // 发送几次心跳包对方未响应则`close`连接
* `TCP`连接最大超时时间为`tcp_keepalive_time + tcp_keepalive_intvl * tcp_keepalive_probes`

#### 5. 参考资料

[LVS: Persistent Connection](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.persistent_connection.html)