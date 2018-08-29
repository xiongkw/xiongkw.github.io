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

#### 3. 集群安装

例如在以下三台机器上安装集群

```
192.168.1.101
192.168.1.102
192.168.1.103
```


##### 3.1 192.168.1.101
修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster              # 集群名称
node.name: node-101                   # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.101           # 当前节点的IP地址
http.port: 9200                       # 对外提供服务的端口

#集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析
discovery.zen.ping.unicast.hosts: ["192.168.1.101","192.168.1.102","192.168.1.103"]

discovery.zen.minimum_master_nodes: 2 # 为了避免脑裂，集群节点数最少为 半数+1
```

##### 3.2 192.168.1.102
修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster              # 集群名称
node.name: node-102                   # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.102           # 当前节点的IP地址
http.port: 9200                       # 对外提供服务的端口

#集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析
discovery.zen.ping.unicast.hosts: ["192.168.1.101","192.168.1.102","192.168.1.103"]

discovery.zen.minimum_master_nodes: 2 # 为了避免脑裂，集群节点数最少为 半数+1
```


##### 3.3 192.168.1.103
修改配置`config/elasticsearch.yaml`

```
cluster.name: es-cluster              # 集群名称
node.name: node-103                   # 节点名称，仅仅是描述名称，用于在日志中区分

network.host: 192.168.1.103           # 当前节点的IP地址
http.port: 9200                       # 对外提供服务的端口

#集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析
discovery.zen.ping.unicast.hosts: ["192.168.1.101","192.168.1.102","192.168.1.103"]

discovery.zen.minimum_master_nodes: 2 # 为了避免脑裂，集群节点数最少为 半数+1
```

##### 3.4 启动集群

分别在每个节点上启动

```
bin/elasticsearch -d -p pid
```

#### 4. 访问

单机环境直接访问即可，集群环境中的任何一个节点都可以直接访问，例如

```
curl 192.168.1.101:9200/_cat/nodes

192.168.1.101 30 32 9 3.99 5.32 5.40 mdi - node-101
192.168.1.102 31 32 9 3.99 5.32 5.40 mdi * node-102
192.168.1.103 31 32 9 3.99 5.32 5.40 mdi - node-103
```

#### 5. 问题

##### 5.1 max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]

需要设置用户可打开文件数

```
$ vi /etc/security/limits.conf

es soft nofile 65536
es hard nofile 65536

```

重新登录生效，需要设置sshd参数

```
$ vi /etc/ssh/sshd_config

UsePAM yes
UseLogin yes
```

##### 5.2 max number of threads [3818] for user [es] is too low, increase to at least [4096]

修改用户最大进程数

```
$ vi /etc/security/limits.conf

es soft nproc 4096
es hard nproc 4096

```

##### 5.3 max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

```
$ vi /etc/sysctl.conf

vm.max_map_count=262144

$ sysctl -p
```

#### 5. 参考

[Installing Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html)