---
layout: post
title: Apache-Storm安装和运维
categories: [编程, java, linux]
tags: [storm]
---


> `Apache Storm`是一个分布式实时大数据处理系统。`Storm`设计用于在容错和水平可扩展方法中处理大量数据。它是一个流数据框架，具有最高的摄取率。虽然`Storm`是无状态的，它通过`Apache ZooKeeper`管理分布式环境和集群状态。它很简单，您可以并行地对实时数据执行各种操作   
> `Apache Storm`继续成为实时数据分析的领导者。`Storm`易于设置和操作，并且它保证每个消息将通过拓扑至少处理一次

#### 1. 简介

#### 2. 安装

到`Storm`官网下载安装包

[http://storm.apache.org/downloads.html](http://storm.apache.org/downloads.html "下载")

##### 2.1 依赖安装

`apache-storm-1.2.1`版本依赖`jdk7`和`zookeeper`，这里省略

> 注意`storm ui`的`Profiling and Debugging`使用到[Java Flight Recorder](http://www.ateam-oracle.com/java-flight-recorder/)，需要安装支持`Java Flight Recorder`的`jdk`   
> `openjdk`不支持

##### 2.2 解压

分别解压到所有主机节点上:

```
tar zxvf apache-storm-1.2.1.tar.gz
```

> 这里安装集群模式的`storm`

##### 2.3 配置

节点`192.168.1.101`

```
# zookeeper servers,注意这里是数组和平常zk的connectString不同，且端口需要另外指定
storm.zookeeper.servers:
     - "192.168.1.100"

#zookeeper端口
storm.zookeeper.port: 8181

#指定本地工作目录
storm.local.dir: "/home/fool/data/storm"

# 指定nimbus节点，可以有多个
nimbus.seeds : ["192.168.1.101"]

# 当前节点名称，这里配置为本机ip
storm.local.hostname: "192.168.1.101"

# ui 服务的端口
ui.port: 8081

# supervisor的工作进程端口
supervisor.slots.ports:
     - 10000
     - 10001
     - 10002
     - 10003
```

节点`192.168.1.102`

```
# zookeeper servers,注意这里是数组和平常zk的connectString不同，且端口需要另外指定
storm.zookeeper.servers:
     - "192.168.1.100"

#zookeeper端口
storm.zookeeper.port: 8181

#指定本地工作目录
storm.local.dir: "/home/fool/data/storm"

# 指定nimbus节点，可以有多个
nimbus.seeds : ["192.168.1.101"]

# 当前节点名称，这里配置为本机ip
storm.local.hostname: "192.168.1.102"

# supervisor的工作进程端口
supervisor.slots.ports:
     - 10000
     - 10001
     - 10002
     - 10003
```

#### 3. 启动

> This command should be run under supervision with a tool like daemontools or monit

`storm`建议使用`daemontools`或`monit`来管理其进程，这里使用`daemontools`

> 参考[Linux下使用daemontools管理进程]({{site.url}}/2018/04/25/linux-daemontools/)

##### 3.1 编写run脚本

这里以`nimbus`为例:

```
$ mkdir nimbus
$ vi nimbus/run

#!/bin/bash

exec storm nimbus >/dev/null 2>&1

$ ln -s nimbus /service/
```

> 分别编写`supervisor ui run`脚本，省略...   

#### 3.2 启动

启动`daemontools svscanboot`就可以了

```
/command/svscanboot &
```

> 注意`svscanboot`会使用默认的`PATH`，所以会出现找不到命令的错误，解决方法参考[Linux下使用daemontools管理进程]({{site.url}}/2018/04/25/linux-daemontools/)

#### 4. 访问ui

浏览器访问

```
http://192.168.1.101:8081/index.html
```

![]({{site.url}}/public/images/2018-04-24-apache-storm-install-01.png)

#### 5. 监控管理

##### 5.1 日志查询

进入`Topology`详情页面，在`Worker Resources`中点击`Port`打开`worker`日志

![]({{site.url}}/public/images/2018-04-24-apache-storm-install-02.png)

> 注意：查看`worker`日志，需要启动`logviewer`

##### 5.2 修改日志等级

进入`Topology`详情页面，点击`Topology actions`中的`Change Log Level`按钮

##### 5.3 debug

需要设置`topology.eventlogger.executors`参数，并重新部署`topology`
```java
conf.setNumEventLoggers(workersNum);
```

进入`Topology`详情页面，点击`Topology actions`中的`Debug`按钮

![]({{site.url}}/public/images/2018-04-24-apache-storm-install-03.png)

在`Component summary`中点击`events`查看`debug`日志

![]({{site.url}}/public/images/2018-04-24-apache-storm-install-04.png)

##### 5.4 jvm性能分析

需要设置`worker.profiler.enabled`参数为`true`，并重新部署`topology`
```java
conf.put(Config.WORKER_PROFILER_ENABLED, true);
```

进程`Bolt`详情页面，在`Executors (All time)`列表中勾选指定`Debug`，然后点击`Profiling and Debugging`中相应按钮

![]({{site.url}}/public/images/2018-04-24-apache-storm-install-05.png)

点击`files`查看和下载`dump`文件

#### 5. 参考文档

* [Apache Storm](http://storm.apache.org/releases/1.2.1/index.html)
* [https://github.com/apache/storm/blob/v1.2.1/conf/defaults.yaml](https://github.com/apache/storm/blob/v1.2.1/conf/defaults.yaml)

