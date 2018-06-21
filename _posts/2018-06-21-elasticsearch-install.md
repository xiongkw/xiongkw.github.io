---
layout: post
title: ElasticSearch安装
categories: [编程, linux]
tags: [ElasticSearch]
---


> Elasticsearch 是一个分布式的 RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况

#### 1. 下载

到官网[下载](https://www.elastic.co/cn/downloads/elasticsearch)安装包，这里以`tar.gz`为例

解压
```
tar zxvf elasticsearch-6.2.4.tar.gz

```

#### 2. 单机版安装

修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster              # 集群名称
node.name: node-101                   # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.101           # 当前节点的IP地址
http.port: 9200                       # 对外提供服务的端口

```

直接启动即可

```
bin/elasticsearch -d -p pid
```

说明：

* `-d`: 以守护进程方式运行
* `-p pid`: 把进程ID写入pid文件

## 4. 集群安装
在如下三台机器上安装集群

```
192.168.1.101
192.168.1.102
192.168.1.103
```


### 4.1 192.168.1.101
修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster                                 # 集群名称
node.name: node-101                                             # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.101                                   # 当前节点的IP地址
http.port: 9200                                             # 对外提供服务的端口

discovery.zen.ping.unicast.hosts: ["192.168.1.101","192.168.1.102","192.168.1.103"] #集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析

discovery.zen.minimum_master_nodes: 2                       # 为了避免脑裂，集群节点数最少为 半数+1
```

### 4.2 192.168.1.102
修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster                                 # 集群名称
node.name: node-102                                             # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.102                                   # 当前节点的IP地址
http.port: 9200                                             # 对外提供服务的端口

discovery.zen.ping.unicast.hosts: ["192.168.1.101","192.168.1.102","192.168.1.103"] #集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析

discovery.zen.minimum_master_nodes: 2                       # 为了避免脑裂，集群节点数最少为 半数+1
```


### 4.3 192.168.1.103
修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster                                 # 集群名称
node.name: node-103                                             # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.103                                   # 当前节点的IP地址
http.port: 9200                                             # 对外提供服务的端口

discovery.zen.ping.unicast.hosts: ["192.168.1.101","192.168.1.102","192.168.1.103"] #集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析

discovery.zen.minimum_master_nodes: 2                       # 为了避免脑裂，集群节点数最少为 半数+1
```

### 4.4 启动集群

分别在每个节点上启动

```
bin/elasticsearch -d -p pid
```

## 5. 访问

单机环境直接访问即可，集群环境中的任何一个节点都可以直接访问，例如

```
curl 192.168.1.101:9200/_cat/nodes

192.168.1.101 30 32 9 3.99 5.32 5.40 mdi - node-101
192.168.1.102 31 32 9 3.99 5.32 5.40 mdi * node-102
192.168.1.103 31 32 9 3.99 5.32 5.40 mdi - node-103
```

## 6. 参考

[Installing Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html)