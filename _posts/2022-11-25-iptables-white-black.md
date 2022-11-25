---
layout: post
title: iptables白名单配置
categories: [编程, linux]
tags: [iptables]
---


> 

#### 1. 默认规则

```
#清空所有默认规则
iptables -F
#清空所有自定义规则
iptables -X
#所有计数器归0
iptables -Z

#先ACCEPT所有
iptables -P INPUT ACCEPT
#开放lo接口的数据包
iptables -A INPUT -i lo -j ACCEPT
#开放22端口，非常重要
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
#允许ping
iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT
#其它INPUT默认DROP
iptables -P INPUT DROP
#开放所有OUTPUT
iptables -P OUTPUT ACCEPT
#开放所有FORWARD
iptables -P FORWARD ACCEPT
```

#### 2. 开放端口

```
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

#### 3. 设置白名单

```
# 设置白名单
iptables -A INPUT -p tcp -s 192.168.1.100 -j ACCEPT
# 解除白名单
iptables -D INPUT -p tcp -s 192.168.1.100 -j ACCEPT
```

#### 4. 设置黑名单

```
#设置黑名单
iptables -I INPUT -s 192.168.0.100 -j DROP
#解除黑名单
iptables -D INPUT -s 192.168.0.100 -j DROP
```

#### 5. 备份和恢复
```
# 导出备份
iptables-save > iptables
# 恢复
iptables-restore < iptables
```

#### 6. 其它

```
# 查看规则
iptables -L [INPUT] -n --line-number
# 删除规则
iptables -D INPUT ${line-number}
```