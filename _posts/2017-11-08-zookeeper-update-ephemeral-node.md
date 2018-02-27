---
layout: post
title: zookeeper临时节点丢失的问题
categories: [编程, java]
tags: [zookeeper]
---

> 微服务架构下用`zookeeper`作为服务注册中心，在服务启动后向`zookeeper`注册服务`ip`和端口信息，发现服务重启时偶尔注册失败

#### 1. 异常现象
日志提示注册成功，也没有异常。经过多次重启发现：
* 重启的时间控制在`session timeout`之内才会出现
* 满足第一个条件的情况下，每隔一次才出现

于是怀疑和`session timeout` 有关，进一步查看节点`session`

#### 2. 异常跟踪

##### 2.1 服务启动且正常注册后，查看节点信息
```
cZxid = 0x132c
ctime = Wed Nov 08 15:53:42 CST 2017
mZxid = 0x132d
mtime = Wed Nov 08 15:53:42 CST 2017
pZxid = 0x132c
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x15f8f0496020037
```

##### 2.2 session timeout时间内重启服务并查看节点信息
```
cZxid = 0x132c
ctime = Wed Nov 08 15:53:42 CST 2017
mZxid = 0x1330
mtime = Wed Nov 08 15:54:07 CST 2017
pZxid = 0x132c
cversion = 0
dataVersion = 3
ephemeralOwner = 0x15f8f0496020037
```
此时节点还存在，并且修改时间变了`mtime = Wed Nov 08 15:54:07 CST 2017`

##### 2.3 session timeout时间过后，发现节点不存在了，查看zookeeper服务端日志
```
2017-11-08 15:54:07,420 [myid:] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@868] - Client attempting to establish new session at /127.0.0.1:59172
2017-11-08 15:54:07,469 [myid:] - INFO  [SyncThread:0:ZooKeeperServer@617] - Established session 0x15f8f0496020038 with negotiated timeout 4000 for client /127.0.0.1:59172
2017-11-08 15:54:10,002 [myid:] - INFO  [SessionTracker:ZooKeeperServer@347] - Expiring session 0x15f8f0496020037, timeout of 4000ms exceeded
2017-11-08 15:54:10,002 [myid:] - INFO  [ProcessThread(sid:0 cport:-1)::PrepRequestProcessor@494] - Processed session termination for sessionid: 0x15f8f0496020037
```
发现`session 0x15f8f0496020037` 过期之前，`0x15f8f0496020038`已经建立了

根据`2.2`和`2.3`得出结论：
* `session 037`过期之前，`session 038`建立连接，并修改了节点数据，此时`session 038`是正常注册到zk的
* `session 037`过期之后，节点被销毁
* 通过`2.1`和`2.2`中`ephemeralOwner`可以看出，节点的所有者一直是`session 037`，`session 038`建立以后并没有改变其所有者

#### 3. 查看服务注册代码
```java
    Stat stat = zookeeper.exists(path, false);
    if (stat == null) {
        zookeeper.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
    } else {
        zookeeper.setData(path, data, -1);
    }
```

问题出在这里：注册服务节点的时候有个判断逻辑，节点不存在则`create`，否则`setData`

#### 4. 修正
修改代码
```java
    Stat stat = zookeeper.exists(path, false);
    if (stat != null) {
        zookeeper.delete(path, -1);
    }
    zookeeper.create(path, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
```
改为先`delete`后`create`，问题解决