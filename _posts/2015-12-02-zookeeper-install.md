---
layout: post
title: Zookeeper安装指南
categories: [编程]
tags: [zookeeper]
---

## 1. Zookeeper介绍
ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。

它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、名字服务、分布式同步、组服务等。

> 参考[Zookeeper官方文档](http://zookeeper.apache.org/)

## 2. Zookeeper下载
从Zookeeper官网下载Zookeeper安装文件zookeeper-3.x.x.tar.gz

> 详情略

## 3. 单机Zookeeper安装
### 3.1 安装
解压安装包到本地硬盘即可
```shell
tar zxvf zookeeper-3.x.x.tar.gz
```

### 3.2 配置
新建conf/zoo.cfg并编辑
```properties
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```
> 可直接复制conf/zoo_sample.cfg为zoo.cfg

参数说明：

> tickTime 心跳间隔时间   
> dataDir 数据存储目录   
> clientPort 服务监听端口

### 3.3 启动
命令行启动

```
bin/zkServer.sh start
```

> 单机版Zookeeper适用于测试场景，生产环境建议安装集群版

## 4. 集群Zookeeper安装

### 4.1 环境要求
集群Zookeeper建议安装3个以上实例

假设三台主机ip分别如下：
```
192.168.1.101
192.168.1.102
192.168.1.103
```

### 4.2 安装
分别解压安装Zookeeper到每台主机

### 4.3 配置

* 修改每个实例的conf/zoo.cfg

```properties
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181

server.1=192.168.1.101:2888:3888
server.2=192.168.1.102:2888:3888
server.3=192.168.1.103:2888:3888
```

* 编辑每个实例的myid文件

192.168.1.101
```
echo 1 > /var/lib/zookeeper/myid
```

192.168.1.102
```
echo 2 > /var/lib/zookeeper/myid
```

192.168.1.102
```
echo 2 > /var/lib/zookeeper/myid
```

> myid文件存放于dataDir目录

### 4.4 启动
分别在命令行启动每个实例

```
bin/zkServer.sh start
```
