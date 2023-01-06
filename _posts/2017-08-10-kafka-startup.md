---
layout: post
title: Kafka入门
categories: [编程, java]
tags: [kafka]
---

> `Kafka`是由`LinkedIn`开源的一个分布式的消息系统，使用`Scala`编写，它以可水平扩展和高吞吐率而被广泛使用，目前由`Apache`维护。

#### 1. Kafka介绍

##### 1.1 角色介绍
* `Broker`：`Kafka`集群包含一个或多个服务实例，每个服务实例就是一个`broker`
* `Topic`：消息主题
* `Partition`：分区，每个`Topic`包含一个或多个`Partition.`
* `Producer`：消息生产者
* `Consumer`：消息消费者
* `Consumer Group`：消费者分组，每个`Consumer`属于一个特定的`Consumer Group`

#### 2. 安装

##### 2.1 环境依赖
* `jdk` 最低`1.7u51`，推荐`1.8`以上
* `zookeeper`

##### 2.2 安装包获取
在[Kafka官网](http://kafka.apache.org/)下载，解压即可

##### 2.3 配置
`config/server.properties`
```properties
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=1

# Switch to enable topic deletion or not, default value is false
delete.topic.enable=true

listeners=PLAINTEXT://192.168.1.100:9092

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=8

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=16

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600


# A comma seperated list of directories under which to store log files
log.dirs=/tmp/kafka-logs

# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=1

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=72

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=1073741824

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

zookeeper.connect=192.168.1.120:2181

# Timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=6000

```

#### 3. 基本命令
```
# 启动
kafka-server-start.sh -daemon ../config/server.properties &
 
# 创建topic
kafka-topics.sh --bootstrap-server 192.168.1.100:9092 --create --topic test --partitions 3 --replication-factor 2

# 查看topic list
kafka-topics.sh --bootstrap-server 192.168.1.100:9092 --list

# 查看topic
kafka-topics.sh --bootstrap-server 192.168.1.100:9092 --topic test --describe

# 删除topic
kafka-topics.sh --bootstrap-server 192.168.1.100:9092 --topic test --delete

# 生产消息，ctrl+c结束
kafka-console-producer.sh --bootstrap-server 192.168.1.100:9092 --topic test

# 消费消息
kafka-console-consumer.sh --bootstrap-server 192.168.1.100:9092 --topic test --from-beginning

# 查看消费组列表
kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --list

# 查看消费组
kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --describe test

# 查看topic消息消费情况
kafka-consumer-offset-checker.sh --bootstrap-server 192.168.1.100:9092 --group test --topic test

# 设置offset到earliest、latest
kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --group test --reset-offsets --topic test --to-earliest --execute

# 设置offset到当前位置（解决offset的异常）
kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --group test --reset-offsets --topic test --to-current --execute
 
# 设置offset到指定位置
kafka-consumer-groups.sh --bootstrap-server 192.168.1.100:9092 --group test --reset-offsets --all-topics --to-offset 500000 --execute
```

#### 4. java client
以下是一个通过`java`客户端访问`kafka`的例子

##### 4.1 maven dependency
```xml
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>0.10.0.1</version>
    </dependency>
```

##### 4.2 producer
```java
    Properties props = new Properties();
    props.put("bootstrap.servers", "192.168.1.100:9092");
    props.put("acks", "1");
    props.put("retries", 0);
    props.put("batch.size", 16384);
    props.put("linger.ms", 1);
    props.put("buffer.memory", 33554432);
    props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
    Producer<String, String> producer = new KafkaProducer<>(props);
    for (int i = 0; i < 100; i++) {
        producer.send(new ProducerRecord<String, String>("test", Integer.toString(i), Integer.toString(i)));
    }
    producer.close();
```
##### 4.3 consumer
```java
    Properties props = new Properties();
    props.put("bootstrap.servers", "192.168.1.100:9092");
    props.put("group.id", "test");
    props.put("enable.auto.commit", "true");
    props.put("auto.commit.interval.ms", "1000");
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
    consumer.subscribe(Arrays.asList("test"));
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records)
            System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
    }
```

#### 5. kafka参数说明

* 详见[Kafka configuration](http://kafka.apache.org/documentation/#configuration)