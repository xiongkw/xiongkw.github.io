---
layout: post
title: zookeeper会话超时分析
categories: [编程, java]
tags: [java, zookeeper, docker]
---

> 微服务架构，服务部署在docker容器中，使用zookeeper实现服务注册和自动发现。在服务的rpc调用中经常发现找不到后端服务

zookeeper版本：
```
客户端: 3.4.6
服务端: 3.4.8
```

打开客户端 zookeeper INFO日志后，发现大量客户端超时现象
```
[0704110251.234] INFO  org.apache.zookeeper.ClientCnxn -Client session timed out, have not heard from server in 2667ms for sessionid 0x35cf3282d980141, closing socket
connection and attempting reconnect
[0704110251.866] INFO  org.apache.zookeeper.ClientCnxn -Opening socket connection to server 10.142.232.139/10.142.232.139:8131
[0704110251.868] INFO  org.apache.zookeeper.ClientCnxn -Socket connection established to 10.142.232.139/10.142.232.139:8131, initiating session
[0704110251.870] INFO  org.apache.zookeeper.ClientCnxn -Session establishment complete on server 10.142.232.139/10.142.232.139:8131, sessionid = 0x35cf3282d980141, neg
otiated timeout = 4000
[0704110253.325] INFO  org.apache.zookeeper.ClientCnxn -Client session timed out, have not heard from server in 2668ms for sessionid 0x15cf32a727f01b0, closing socket
connection and attempting reconnect
[0704110254.046] INFO  org.apache.zookeeper.ClientCnxn -Opening socket connection to server 10.142.232.140/10.142.232.140:8131
[0704110254.048] WARN  org.apache.zookeeper.ClientCnxn -Session 0x15cf32a727f01b0 for server null, unexpected error, closing socket connection and attempting reconnect
java.net.ConnectException: Connection refused
        at sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)
        at sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:744)
        at org.apache.zookeeper.ClientCnxnSocketNIO.doTransport(ClientCnxnSocketNIO.java:361)
        at org.apache.zookeeper.ClientCnxn$SendThread.run(ClientCnxn.java:1081)
```
zookeeper服务端日志
```
2017-07-04 11:02:59,645 [myid:1] - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:8131:NIOServerCnxn@357] - caught end of stream exception
EndOfStreamException: Unable to read additional data from client sessionid 0x0, likely client has closed socket
        at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:228)
        at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:203)
        at java.lang.Thread.run(Thread.java:745)
```

首先怀疑和zookeeper版本有关，于是搭建一个3.4.6的zk服务端，做一个比对测试

|zk版本   |   不做rpc调用24小时  |  小报文(1k)50并发间隔1s 12小时 | 大报文(1M)20并发间隔1s 1分钟 |
| ------- | -------------------- | ------------------------------ | ------------------------ |
|3.4.8    |          20          |               大量             |          大量            |   
|3.4.6    |           0          |                 0              |          大量           |

得出结论：
* 和zk版本有一定关系，使用配套的版本会更稳定
* 和网络有关，网络繁忙时两个版本都会出现大量超时