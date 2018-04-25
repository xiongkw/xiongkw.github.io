---
layout: post
title: Apache-Storm安装
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

##### 3.1 编写启停脚本

`startup.sh`

```
#!/bin/bash

# 启动nimbus
nohup ./storm nimbus >/dev/null 2>&1 &
# 启动 supervisor
nohup ./storm supervisor >/dev/null 2>&1 &
# 启动 ui
nohup ./storm ui >/dev/null 2>&1 &
```

`shutdown.sh`

    #!/bin/bash

    # 停止 nimbus
    kill -9 `ps aux|grep nimbus|grep -v grep|awk '{print $2}'`
    # 停止 supervisor
    kill -9 `ps aux|grep supervisor|grep -v grep|awk '{print $2}'`
    # 停止 ui
    kill -9 `ps aux|grep storm\.ui\.core|grep -v grep|awk '{print $2}'`

> 上面的启停脚本包含了`nimbus,supervisor,ui`三个角色，实际情况按需提供

#### 3.2 启动

分别在各节点执行启动脚本

```
./startup.sh
```

#### 4. 查看界面

浏览器打开

```
http://192.168.1.101:8081/index.html
```

![]({{site.url}}/public/images/2018-04-24-apache-storm-install.png)

#### 5. 参考文档

* [Apache Storm](http://storm.apache.org/releases/1.2.1/index.html)
* [https://github.com/apache/storm/blob/v1.2.1/conf/defaults.yaml](https://github.com/apache/storm/blob/v1.2.1/conf/defaults.yaml)

