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
Topic分区数设置为128
```

> 两台`Logstash`消费线程数一共为`128`

#### 2. Logstash

##### 2.1 硬件
```
两台64核128G主机
```

##### 2.2 jvm heap

jvm.options
```
-Xms32g
-Xmx32g
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

设置`kafka`消费线程数为`cpu核心数`

```
input {
    kafka {
        consumer_threads => 64
        #...
    }
}
```

使用`dissect`代替`grok`

```
filter {
    dissect {
        mapping => ""
    }
}
```

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

> 设置分片数为`32`(根据总量决定，每个分片大小在`32G`)
> 刷盘时间为`60s`(默认`1s`)

#### 3. ElasticSearch

##### 3.1 硬件

```
三台64核128G集群
```

##### 3.3 jvm heap

jvm.options
```
-Xms32g
-Xmx64g
```

##### 3.3 elasticsearch.yml

在`Kibana`中发现大量`Bulk Rejections`，解决办法是调整`bulk队列`大小

```yaml

# bulk队列
thread_pool.bulk.queue_size: 1000

# 减少重启时分片恢复时间
gateway.recover_after_nodes: 3
```

#### 4. Logstash GC问题

在`Kibana`中监控到`Logstash`处理速率断断续续为0，而`Kafka`中仍有大量消息堆积，而且`GC`比较频繁，进一步分析，发现原因是`Full GC`时间过长导致线程卡死

调整`pipeline.batch.size`和`jvm heap`

logstash.yml
```yaml
pipeline.batch.size: 800
```

jvm.options
```
-Xmn20g
-Xms32g
-Xmx32g
```

#### 5. 性能

##### 5.1 Logstash
```
单台Logstash处理速率可达3.5w/s
```

##### 5.2 ElasticSearch
```
三台ES集群，有备份(1)的情况下索引速率可达4w/s
无备份的情况下索引速率可达7w/s
```
