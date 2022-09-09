---
layout: post
title: Kafka性能调优
categories: [java]
tags: []
---

>  

#### 1. broker

##### 1.1 使用Kraft替换Zookeeper

Kafka从2.8版本开始提供了集成的Kraft组件，用于替换Zookeeper

* Enables Kafka clusters to scale to millions of partitions through improved control plane performance with the new metadata management
* Improves stability, simplifies the software, and makes it easier to monitor, administer, and support Kafka.
* Allows Kafka to have a single security model for the whole system
* Provides a lightweight, single process way to get started with Kafka
* Makes controller failover near-instantaneous


##### 1.2 核心参数

```
# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600
```

##### 1.3 压缩

```
# 开启压缩，和producer保持相同
compression.type=lz4
```

##### 1.4 max.message.bytes

比较奇葩的参数，看上去是指消息体的最大字节数，实际是指批量消息的最大数量，可配置到broker或topic

> 参考[max.message.bytes](https://kafka.apache.org/documentation/#topicconfigs_max.message.bytes)


#### 2. producer

##### 2.1 acks

* 0: 写消息不需要确认，性能最高，但是可能丢数据。
* 1: 只需要Leader确认
* all: 需要所有Leader和Flower确认

##### 2.2 compression.type

```
# 开启压缩，和producer保持相同
compression.type=lz4
```

##### 2.3 批量发送

```
# 批量发送消息的最大时间间隔，和batch.size二者满足一个即发送
linger.ms=100

# 批量消息的最大数量，和linger.ms二者满足一个即发送
batch.size=1024

# 最大请求体字节数
max.request.size

# 缓冲区大小，决定批量消息的最大字节数
buffer.memery=268435456

# 生产者在收到服务器响应之前可以发送消息的数量，设为1可保证消息是按顺序发送
max.in.flight.requests.per.connection=1
```

#### 3. consumer

##### 3.1 核心参数

```
# 两次poll的最大间隔时间
max.poll.interval.ms=30000

# 手动commit offset
enable.auto.commit=false

# 一次拉取消息的最大字节数
fetch.max.bytes

# 一次拉取消息的最大条数
max.poll.records

# 请求超时时间
request.timeout.ms=30000

```

##### 3.2 线程数

由于一个partition同一时刻只能被一个消费线程消费，所以一般情况下：消费线程数等于partition时性能最优

##### 3.3 poll方法

```
# 一次拉取的最大等待时间，和max.poll.records一起使用，二者只要满足其一就返回
poll(1000)
```

#### 参考

* [CONFIGURATION](https://kafka.apache.org/documentation)