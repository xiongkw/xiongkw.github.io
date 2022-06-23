---
layout: post
title: 通过iptables实现端口转发
categories: [linux]
tags: []
---

>  

#### 1. 需求和现状

三台主机通过弹性IP访问，A到B通、B到C通、A到C不通，要求A能访问C的指定端口

```
+----------------+        +----------------+         +----------------+
|       A        |        |        B       |         |       C        |
| 10.50.206.100  | --√--> | 10.50.206.101  | --√-->  | 10.150.90.200  |
| 172.16.200.100 |        | 172.16.200.101 |         | 172.16.200.200 |
+----------------+        +----------------+         +----------------+
        |                                                   |
        +-------------------------×-------------------------+

```


#### 2. 思路

在B上通过iptables做端口转发


#### 3. 方法

##### 3.1 前置条件

先在B上开启ip转发
```
vi /etc/sysctl.conf
net.ipv4.ip_forward =1
sysctl -p
```

##### 3.2 添加DNAT规则

在B主机对所有9098端口的访问转发到C主机的9098端口

```
$ iptables -t nat -A PREROUTING -p tcp --dport 9096 -j DNAT --to 10.150.90.200:9098
```


在A主机访问B主机9098端口

```
$ curl 10.50.206.101:9098
No route to host
```

同时在B主机抓包
```
$ tcpdump -i eth0 port 9098 -nnevvv
10:08:17.294703 fa:16:3e:d0:ae:77 > fa:16:3e:14:4c:64, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 59, id 34438, offset 0, flags [DF], proto TCP (6), length 60)
    10.50.206.100.60618 > 172.16.200.101.9098: Flags [S], cksum 0xf3f8 (correct), seq 3335041735, win 28200, options [mss 1410,sackOK,TS val 1210856513 ecr 0,nop,wscale 7], length 0
```


##### 3.3 添加FORWARD规则

查看B主机FORWARD链规则

```
$ iptabels -L FORWARD -n
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

发现所有转发都被reject了，添加一条规则放开9098端口
```
$ iptables -I FORWARD -p tcp --dport 9098 -j ACCEPT
```

再次curl并抓包

```
10:07:24.472120 fa:16:3e:d0:ae:77 > fa:16:3e:14:4c:64, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 59, id 17209, offset 0, flags [DF], proto TCP (6), length 60)
    10.50.206.100.60576 > 172.16.200.101.9098: Flags [S], cksum 0x8f08 (correct), seq 2251237075, win 28200, options [mss 1410,sackOK,TS val 1210803690 ecr 0,nop,wscale 7], length 0
10:07:24.472201 fa:16:3e:14:4c:64 > fa:16:3e:d0:ae:77, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 58, id 17209, offset 0, flags [DF], proto TCP (6), length 60)
    10.50.206.100.60576 > 10.150.90.200.9098: Flags [S], cksum 0x95fb (correct), seq 2251237075, win 28200, options [mss 1410,sackOK,TS val 1210803690 ecr 0,nop,wscale 7], length 0
```

发现目标地址已经改为C主机弹性IP了，但是这个包没发出去，原因是源地址还是A主机，所以包被丢弃了

##### 3.4 添加SNAT规则

在B主机对所有10.150.90.200:9098的包做SNAT，注意这里的source地址为B主机内网地址

```
$ iptables -t nat -A POSTROUTING -d 10.150.90.200 -p tcp --dport 9098 -j SNAT --to-source 172.16.200.101
```

再次curl并抓包
```
10:13:28.313635 fa:16:3e:d0:ae:77 > fa:16:3e:14:4c:64, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 59, id 11826, offset 0, flags [DF], proto TCP (6), length 60)
    10.50.206.100.60876 > 172.16.200.101.9098: Flags [S], cksum 0x34f4 (correct), seq 2609175838, win 28200, options [mss 1410,sackOK,TS val 1211167532 ecr 0,nop,wscale 7], length 0
10:13:28.313684 fa:16:3e:14:4c:64 > fa:16:3e:d0:ae:77, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 58, id 11826, offset 0, flags [DF], proto TCP (6), length 60)
    172.16.200.101.60876 > 10.150.90.200.9098: Flags [S], cksum 0x329b (correct), seq 2609175838, win 28200, options [mss 1410,sackOK,TS val 1211167532 ecr 0,nop,wscale 7], length 0
10:13:28.314352 fa:16:3e:d0:ae:77 > fa:16:3e:14:4c:64, ethertype IPv4 (0x0800), length 74: (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.50.206.200.9098 > 172.16.200.101.60876: Flags [S.], cksum 0x01c6 (correct), seq 83497963, ack 2609175839, win 27960, options [mss 1410,sackOK,TS val 1211355291 ecr 1211167532,nop,wscale 7], length 0
```

可以看到DNAT后的包已经发到C主机，并且也收到了C主机的ACK包，但是回包并没有发回A主机，原因是被FORWARD链reject了


##### 3.5 添加FORWARD规则


添加一条规则放开源地址为10.150.90.200的包
```
$ iptables -I FORWARD -s 10.150.90.200 -p tcp -j ACCEPT
```

再次curl，发现能成功返回了


