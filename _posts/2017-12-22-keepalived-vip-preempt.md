---
layout: post
title: keepalived主备机切换时的抢占问题
categories: [编程, linux]
tags: [lvs, keepalived, vip, vrrp]
---

> 通过代理程序监听`Real Server`启停状态并刷新`Keepalived`配置，在`Keepalived restart`时，可能会引起主备切换

#### 1. 正常的主备模式
MASTER
```
vrrp_instance V1{
    state MASTER
    priority 101
}
```
BACKUP
```
vrrp_instance V1{
    state BACKUP
    priority 101
}
```
*注意: `MASTER`和`BACKUP`的区分由`priority`决定，与state的定义无关*

> 主备模式下，只要主健康，便会抢占`vip`

#### 2. 双备模式
避免了因主宕机恢复引起的频繁`vip`抢占
MASTER
```
vrrp_instance V1{
    state BACKUP
    priority 100
}
```
BACKUP
```
vrrp_instance V1{
    state BACKUP
    priority 100
}
```
> 双备机且优先级相同的情况下，不会发生`vip`抢占，但有一特殊情况见下文

#### 3. 双备模式下动态刷新`keepalived`配置引起的`vip`漂移

##### 3.1 现象
通过`restart keepalived`模拟，偶尔会发生`vip`漂移现象

##### 3.2 Master_Down_Interval
* Advertisement_Interval: Time interval between ADVERTISEMENTS(seconds). Default is 1 second.

* Master_Down_Interval: Time interval for Backup to declare Master down (seconds). Calculated as:
```
 (3 * Advertisement_Interval) + ( (256 - Priority) / 256 )
```

`MASTER`会定时在局域网内发送`VRRP`通告，`BACKUP`在`Master_Down_interval`时间过后没有收到便会启动新的选举

##### 3.3 为什么重启MASTER的keepalived服务后，BACKUP会马上成为新的MASTER而不用等待`Master_Down_interval`

`MASTER keepalived`重启时会发出一个优先级为0的`ADVERTISEMENT`，表明其释放了`MASTER`，而`BACKUP`收到后便会启动选举定时器：

> If the Priority in the ADVERTISEMENT is Zero, then: Set the Master_Down_Timer to Skew_Time

##### 3.4 由程序动态刷新`keepalived`配置后引起`vip`漂移
程序动态刷新`keepalived`配置的实现方式是写入新的`keepalived`配置后再重启服务，谁会成为`MASTER`取决于刷新的先后顺序，即先`restart`的会成为`MASTER`。

#### 4. 特殊情况
##### 4.1 现象
生产环境的异常演练出现：双备模式(如2)下，拔掉`A(MASTER)`网线后，`vip`漂移到`B(BACKUP)`，重新插上`A`网线后`vip`又漂移到`A`

可见，在插上网线后`A`抢占了`MASTER`

##### 4.2 模拟测试
使用`ifconfig down`和`service keepalived stop`的方式模拟`MASTER`宕机的情况都无法重现

> 因为测试环境的问题，无法直接插拔网线，并且缺少本地主机，所以这里只是简单的模拟。

##### 4.3 VRRP协议
`VRRP`协议里有一段：
> If the Priority in the ADVERTISEMENT is greater than the local Priority,
 or
 If the Priority in the ADVERTISEMENT is equal to the local Priority and the primary IP Address of the sender is greater than the local primary IP Address, then:
 o Cancel Adver_Timer
 o Set Master_Down_Timer to Master_Down_Interval
 o Transition to the {Backup} state
 
说明`priority`并非`MASTER`抢占的唯一条件，如果对方的`ip`大于自己，则仍会被对方抢占

查看`A`的`ip`，确实是大于`B`

##### 4.4 猜测

猜测网线拔掉后，原`BACKUP`没有收到`ADVERTISEMENT`而经过`Master_Down_Interval`后抢占成为`MASTER`，这时再插上网线，原`MASTER`仍然会发出`ADVERTISEMENT`，刚好原`MASTER`的ip大，所以原`MASTER`又抢占成功了。

这里的重点是网线拔掉再插上后，原`MASTER`是否还会发出`ADVERTISEMENT`，而可以确定的是模拟测试`ifconfig down up`后原`MASTER`不会再发出`ADVERTISEMENT`

##### 4.5 验证

> 因为懒得找机器安装linux系统和搭建`MASTER BACKUP`环境做真实的 `网线插拔` 异常演练，这里仅根据以上现象和参考资料做一个大胆的猜测，日后如有机会再做验证

```java
// TODO
```

#### 5. 参考文档
[Virtual Router Redundancy Protocol (VRRP)](https://tools.ietf.org/html/rfc3768)