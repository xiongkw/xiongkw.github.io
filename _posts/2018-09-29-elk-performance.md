---
layout: post
title: elk日志收集系统性能优化
categories: [编程, java, linux]
tags: [elk, elasticsearch, logstash, kibana, kafka, filebeat]
---


> 业务系统每天日志量为20亿，占用索引空间1T，高峰时可达40亿

#### 1. Kafka

##### 1.1 硬件

```
两台64核128G集群
```
##### 1.2 jvm heap

```
KAFKA_HEAP_OPTS="-Xms32G -Xmx64G"
```

##### 1.3 配置

`server.properties`
```
# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=65

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=128

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=10485760

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=10485760

# The number of messages to accept before forcing a flush of data to disk
#log.flush.interval.messages=1000000

# The maximum amount of time a message can sit in a log before we force a flush
#log.flush.interval.ms=60000
```

##### 1.4 Topic

```
Topic分区数设置为192
```

> 三台logstash消费线程数一共为192

#### 2. Logstash

##### 2.1 硬件
```
三台64核128G主机
```

##### 2.2 jvm heap

jvm.options
```
-Xms32g
-Xmx64g
```

##### 2.3 logstash.yml

```yaml
# This defaults to the number of the host's CPU cores.
#
pipeline.workers: 65
#
# How many events to retrieve from inputs before sending to filters+workers
#
pipeline.batch.size: 2500
```

##### 2.4 pipeline

```
input {
    kafka {
        consumer_threads => 64
        #...
    }
}
```

> 设置kafka消费线程数为cpu核心数

##### 2.5 索引模板

```json
{
    "settings" : {
        "index" : {
            "number_of_shards" : 32,
            "number_of_replicas" : 0 ,
            "refresh_interval" : "60s"
        }
    }
}
```

> 设置分片数为32(根据总量决定，每个分片大小在32G)
> 刷盘时间为60s(默认1s)

#### 3. ElasticSearch

##### 3.1 硬件

```
三台64核512G集群
```

##### 3.3 jvm heap

jvm.options
```
-Xms32g
-Xmx64g
```

##### 3.3 elasticsearch.yml

```yaml

# bulk队列
thread_pool.bulk.queue_size: 1000

# 减少重启时分片恢复时间
gateway.recover_after_nodes: 3
```

#### 4. 性能

##### 4.1 Logstash
```
单台Logstash处理速率为2w/s
```

##### 4.2 ElasticSearch
```
三台ES索引速率可达5w/s
```
