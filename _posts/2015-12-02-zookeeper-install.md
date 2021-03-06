---
layout: post
title: Zookeeper安装指南
categories: [编程, java]
tags: [zookeeper]
---

#### 1. Zookeeper介绍
`ZooKeeper`是一个分布式的，开放源码的分布式应用程序协调服务，是`Google`的`Chubby`一个开源的实现，是`Hadoop`和`Hbase`的重要组件。

它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、名字服务、分布式同步、组服务等。

> 参考[Zookeeper官方文档](http://zookeeper.apache.org/)

#### 2. Zookeeper下载
从`Zookeeper`官网下载`Zookeeper`安装文件`zookeeper-3.x.x.tar.gz`

> 详情略

#### 3. 单机Zookeeper安装
##### 3.1 安装
解压安装包到本地硬盘即可
```shell
tar zxvf zookeeper-3.x.x.tar.gz
```

##### 3.2 配置
新建`conf/zoo.cfg`并编辑
```properties
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
```
> 可直接复制`conf/zoo_sample.cfg`为`zoo.cfg`

参数说明：

* `tickTime` 心跳间隔时间
* `dataDir` 数据存储目录   
* `clientPort` 服务监听端口   
* `autopurge.snapRetainCount` 保留快照文件数   
* `autopurge.purgeInterval` 清除快照文件时间间隔(h)

##### 3.3 启动
命令行启动

```
bin/zkServer.sh start
```

> 单机版`Zookeeper`适用于测试场景，生产环境建议安装集群版

#### 4. 集群Zookeeper安装

##### 4.1 环境要求
集群`Zookeeper`建议安装3个以上实例

假设三台主机ip分别如下：
```
192.168.1.101
192.168.1.102
192.168.1.103
```

##### 4.2 安装
分别解压安装`Zookeeper`到每台主机

##### 4.3 配置

* 修改每个实例的`conf/zoo.cfg`

```properties
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181

server.1=192.168.1.101:2888:3888
server.2=192.168.1.102:2888:3888
server.3=192.168.1.103:2888:3888
```

* 编辑每个实例的`myid`文件

`192.168.1.101`
```
echo 1 > /var/lib/zookeeper/myid
```

`192.168.1.102`
```
echo 2 > /var/lib/zookeeper/myid
```

`192.168.1.102`
```
echo 2 > /var/lib/zookeeper/myid
```

> `myid`文件存放于`dataDir`目录

##### 4.4 启动
分别在命令行启动每个实例

```
bin/zkServer.sh start
```

#### 5. 服务和自动启动

编写服务脚本`/etc/init.d/zookeeper`

```sh
#!/bin/bash
#chkconfig:2345 20 90
#description:zookeeper
#processname:zookeeper
ZOO_HOME=/app/zookeeper-3.4.10
export ZOO_LOG_DIR=$ZOO_HOME/logs
case $1 in
        start) su zookeeper $ZOO_HOME/bin/zkServer.sh start;;
        stop) su zookeeper $ZOO_HOME/bin/zkServer.sh stop;;
        status) su zookeeper $ZOO_HOME/bin/zkServer.sh status;;
        restart) su zookeeper $ZOO_HOME/bin/zkServer.sh restart;;
        *) echo "require start|stop|status|restart" ;;
esac
```

设置开机启动
```sh
$ chmod 755 zookeeper
$ chkconfig zookeeper on
$ service zookeeper start
```