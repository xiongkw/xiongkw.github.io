---
layout: post
title: Spring Cloud微服务中的调用链分析和统计
categories: [编程, java, spring]
tags: [spring cloud, sleuth, zipkin, trace, kafka, storm, elasticsearch]
---


> 基于`Spring Cloud Sleuth`的微服务调用链分析和统计

#### 1. 设计

![]({{site.url}}/public/images/2018-05-15-spring-cloud-log-and-trace.png)

#### 2. Spring Sleuth

> 微服务架构下，服务之间调用会非常频繁，为了避免高峰时段大量调用链压跨日志收集服务，这里使用`kafka`做缓冲

##### 2.1 配置zipkin发送调用链到kafka
`pom.xml`
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

`application.yml`
```yaml
spring:
    sleuth:
      sampler:
        percentage: 1
    zipkin:
          kafka:
            topic: zipkin
    kafka:
      bootstrap-servers: 192.168.1.101:9092,192.168.1.102:9092
```

> `spring.sleuth.sampler.percentage=1` 采样率，这里设为`1`，即全量采样   
> `spring.zipkin.kafka.topic=test`，指定发送到`kafka`的`topic`

#### 3. Kafka

##### 3.1 调用链消息格式

以下是三个服务调用时，`kafka`收集到的消息格式

```json
[
    {
        "traceId":"d7826ea4f43bcf6a",
        "parentId":"68105b34c7b3356c",
        "id":"0d8a0b5c6236860a",
        "kind":"SERVER",
        "name":"http:/users/2",
        "timestamp":1524563180063000,
        "duration":2381,
        "localEndpoint":{
            "serviceName":"data",
            "ipv4":"172.26.116.215",
            "port":8777
        },
        "tags":{
            "mvc.controller.class":"DataController",
            "mvc.controller.method":"getById",
            "spring.instance_id":"data-2528536179-bhv58:data:8777"
        },
        "shared":true
    }
]
```

```json
[
    {
        "traceId":"d7826ea4f43bcf6a",
        "parentId":"d7826ea4f43bcf6a",
        "id":"68105b34c7b3356c",
        "kind":"CLIENT",
        "name":"http:/users",
        "timestamp":1524563183832000,
        "duration":16000,
        "localEndpoint":{
            "serviceName":"member",
            "ipv4":"172.26.230.150",
            "port":8777
        },
        "tags":{
            "http.host":"172.26.155.31",
            "http.method":"GET",
            "http.path":"/users",
            "http.url":"http://172.26.155.31:8777/users?id=2",
            "spring.instance_id":"member-2967759577-jfj57:member:8777"
        }
    },
    {
        "traceId":"d7826ea4f43bcf6a",
        "id":"d7826ea4f43bcf6a",
        "kind":"SERVER",
        "name":"http:/members/2",
        "timestamp":1524563183831000,
        "duration":22836,
        "localEndpoint":{
            "serviceName":"member",
            "ipv4":"172.26.230.150",
            "port":8777
        },
        "tags":{
            "http.host":"member.aepdev.com",
            "http.method":"GET",
            "http.path":"/members/2",
            "http.status_code":"200",
            "http.url":"http://172.26.230.150:8777/members/2",
            "mvc.controller.class":"MemberController",
            "mvc.controller.method":"getMember",
            "spring.instance_id":"member-2967759577-jfj57:member:8777"
        }
    }
]
```

```json
[
    {
        "traceId":"d7826ea4f43bcf6a",
        "parentId":"68105b34c7b3356c",
        "id":"0d8a0b5c6236860a",
        "kind":"CLIENT",
        "name":"http:/users/2",
        "timestamp":1524563185091000,
        "duration":6000,
        "localEndpoint":{
            "serviceName":"user",
            "ipv4":"172.26.155.31",
            "port":8777
        },
        "tags":{
            "http.host":"172.26.116.215",
            "http.method":"GET",
            "http.path":"/users/2",
            "http.url":"http://172.26.116.215:8777/users/2",
            "spring.instance_id":"user-2109957813-buet4:user:8777"
        }
    },
    {
        "traceId":"d7826ea4f43bcf6a",
        "parentId":"d7826ea4f43bcf6a",
        "id":"68105b34c7b3356c",
        "kind":"SERVER",
        "name":"http:/users",
        "timestamp":1524563185089000,
        "duration":13354,
        "localEndpoint":{
            "serviceName":"user",
            "ipv4":"172.26.155.31",
            "port":8777
        },
        "tags":{
            "mvc.controller.class":"UserController",
            "mvc.controller.method":"getUser",
            "spring.instance_id":"user-2109957813-buet4:user:8777"
        },
        "shared":true
    }
]

```

> 可以看到同一次调用被分为了两个`span`记录(`CLIENT`和`SERVER`)

#### 4. Storm

> `storm`的作用是采集`span`，合并相同`ID`的`span`，优化`span`的数据结构，以便于`es`处理

##### 4.1 storm代码

```jave
BrokerHosts hosts = new ZkHosts("192.168.1.100:2181");
SpoutConfig spoutConfig = new SpoutConfig(hosts, "zipkin", "/zipkin", TRACE_LOG_TOPOLOGY);

EsConfig esConfig = new EsConfig(new String[]{"http://192.168.1.100:8200"});

spoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());
spoutConfig.zkServers = Arrays.asList("192.168.1.100");
spoutConfig.zkPort = 2181;
KafkaSpout kafkaSpout = new KafkaSpout((spoutConfig));

TopologyBuilder builder = new TopologyBuilder();
String kafkaSpoutId = "kafkaSpout";
builder.setSpout(kafkaSpoutId, kafkaSpout, 2);
String collectionBoltId = "spanCollectionBolt";
builder.setBolt(collectionBoltId, new SpanCollectionBolt(), 2).shuffleGrouping(kafkaSpoutId);

String mergingBoltId = "spanMergingBolt";
// 根据spanId分组，让相同spanId的span发送到同一个SpanMergingBolt
builder.setBolt(mergingBoltId, new SpanMergingBolt(), 2).fieldsGrouping(collectionBoltId, new Fields("spanId"));

String namingBoltId = "spanNamingBolt";
// 根据traceId分组，让相同traceId的span发送到同一个SpanNamingBolt
builder.setBolt(namingBoltId, new SpanNamingBolt(), 2).fieldsGrouping(mergingBoltId, new Fields("traceId"));

String storingBoltId = "spanStoringBolt";
builder.setBolt(storingBoltId, new SpanStoringBolt(esConfig), 2).shuffleGrouping(namingBoltId);
```

##### 4.2 SpanCollectionBolt

> `SpanCollectionBolt`的作用是消费`kafka`中`ZipkinSpan`数据，提取并优化为业务需要的数据结构

```java
public void execute(Tuple input) {
    String value = (String) input.getValue(0);
    // spring sleuth 发送到kafka的消息格式为ZipkinSpan数组
    ZipkinSpan[] zipkinSpans = JsonUtils.parseSilently(value, ZipkinSpan[].class);

    for (ZipkinSpan zipkinSpan : zipkinSpans) {
        // 转换为符合业务格式的span数据结构
        Span span = toSpan(zipkinSpan);
        if (span == null) {
            continue;
        }
        collector.emit(input, new Values(span.getId(), span));
    }
    collector.ack(input);
}

public void declareOutputFields(OutputFieldsDeclarer declarer) {
    // 定义输出字段spanId和span
    declarer.declare(new Fields("spanId", "span"));
}
```

##### 4.3 SpanMergingBolt

> `SpanMergingBolt`的作用是合并相同`ID`的`span`片段为一个完整的`span`(因为`zipkin`发送到`kafka`的`span`只是调用中的一个片段)

````java
public void execute(Tuple input) {
    String spanId = (String) input.getValueByField("spanId");
    Span span = (Span) input.getValueByField("span");
    boolean isEntranceSpan = span.getParentId() == null;
    // 入口span只有一个片段，不需要合并
    if (!isEntranceSpan) {
        Holder<Tuple> holder = tupleMap.get(spanId);
        if (holder == null) {
            tupleMap.put(spanId, new Holder(input));
            return;
        }
        Tuple t = holder.get();
        merge(span, (Span) t.getValueByField("span"));
        tupleMap.remove(spanId);
        collector.ack(t);
    }
    postProcessSpan(span);
    collector.emit(input, new Values(span.getTraceId(), span));
    collector.ack(input);
}

private void merge(Span span, Span s) {
    BeanUtils.copyNonNullProperties(s, span);
    postProcessSpan(span);
}

@Override
public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("traceId", "span"));
}
````

##### 4.4 SpanNamingBolt

> `SpanNamingBolt`的作用是给同一`traceId`相同的`span`赋上`traceName`(入口`span`的`name`)

```java
public void execute(Tuple input) {
    Span span = (Span) input.getValueByField("span");
    String traceId = span.getTraceId();
    boolean isEntranceSpan = span.getParentId() == null;
    if (!isEntranceSpan) {
        Holder<String> traceNameHolder = traceNameMap.get(traceId);
        if (traceNameHolder != null) {
            span.setTraceName(traceNameHolder.get());
            collector.emit(input, new Values(traceId, span.getClientIp() + span.getServerIp(), span));
            collector.ack(input);
            return;
        }
        Holder<Map<String, Tuple>> holder = traceSpanMap.get(traceId);
        if (holder == null) {
            holder = new Holder(new ConcurrentHashMap<>(1024));
            traceSpanMap.put(traceId, holder);
        }
        holder.get().put(span.getId(), input);
        return;
    }
    String traceName = span.getTraceName();
    traceNameMap.put(traceId, new Holder(traceName));
    postProcessSpanTraceName(traceId, traceName);
    collector.emit(input, new Values(traceId, span.getClientIp() + span.getServerIp(), span));
    collector.ack(input);
}

private void postProcessSpanTraceName(String traceId, String traceName) {
    Holder<Map<String, Tuple>> holder = traceSpanMap.get(traceId);
    if (holder == null) {
        return;
    }
    Map<String, Tuple> spanMap = holder.get();
    Iterator<Map.Entry<String, Tuple>> iterator = spanMap.entrySet().iterator();
    while (iterator.hasNext()) {
        Map.Entry<String, Tuple> next = iterator.next();
        Tuple tuple = next.getValue();
        Span s = (Span) tuple.getValueByField("span");
        s.setTraceName(traceName);
        spanMap.remove(next.getKey());
        collector.emit(tuple, new Values(traceId, s.getClientIp() + s.getServerIp(), s));
        collector.ack(tuple);
    }
}
```

##### 4.5 SpanStoringBolt

> `SpanStoringBolt`的作用是根据一定规则把完整的`span`写入`ElasticSearch`指定索引

```java
protected void process(Tuple tuple) {
    Span span = (Span) tuple.getValueByField("span");
    if (span != null) {
        String str = JsonUtils.toStringSilently(span);
        if (str != null) {
            try {
                client.performRequest("put", getEndpoint(span), EMPTY, new StringEntity(str, ContentType.APPLICATION_JSON));
            } catch (IOException e) {
                collector.reportError(e);
                collector.fail(tuple);
            }
        }
    }
    collector.ack(tuple);
}
```

#### 5. ElasticSearch

##### 5.1 trace查询

我们用`入口span`表示一个`trace`，所以查询`trace`需要两步：

###### 5.1.1 根据条件查询`span`的`traceId`:

```json
{
  "size" : 0,
  "query" : {
    "bool": {
      "must":[
        {"bool":{"should":[{"term":{"serverApp":"${app}"}}, {"term":{"clientApp":"${app}"}}]}}
        ,{"bool":{"should":[{"term":{"server":"${service}"}}, {"term":{"client":"${service}"}}]}}
        ,{"term":{"status":"${status}"}}
        ,{"match":{"error":"${error}"}}
        ,{"bool":{"should":[{"term":{"serverIp":"${ip}"}}, {"term":{"clientIp":"${id}"}}]}}
      ]
    }
  },
  "aggs":{
    "traceIds":{
      "terms":{
        "field":"traceId"
      }
    }
  }
}
```

###### 5.1.2 根据上一步的`traceId`查trace

```json
{
  "query": {
    "bool":{
      "filter":[{
        "terms":{
          "traceId":[
            "ede0a59ec2cdd962","ede0a59ec2cdd962","8842d7f77dcf8759"
          ]
        }
      },{
        "bool":{
          "must_not":{
            "exists":{
              "field":"parentId"
            }
          }
        }
      }]
    }
  }
}
```

##### 5.1 trace统计

统计1小时内所有服务的调用情况

```json
{
  "query": {
    "range": {
      "timestamp": {
        "gte": 1524563180063,
        "lt": 1524566780063
      }
    }
  },
  "size": 0,
  "aggs": {
    "traces": {
      "terms": {
        "field": "traceName"
      },
      "aggs": {
        "spans": {
          "terms": {
            "field": "name"
          },
          "aggs": {
            "client": {
              "terms": {
                "field": "client",
                "script": {
                  "source": "_value",
                  "lang": "painless"
                }
              }
            },
            "server": {
              "terms": {
                "field": "server",
                "script": {
                  "source": "_value",
                  "lang": "painless"
                }
              }
            },
            "timeArea": {
              "histogram": {
                "field": "timestamp",
                "interval": 60000000
              },
              "aggs": {
                "duration": {
                  "sum": {
                    "field": "duration"
                  }
                },
                "times": {
                  "cardinality": {
                    "field": "id"
                  }
                },
                "errors": {
                  "filter": {
                    "match": {
                      "status": "err"
                    }
                  },
                  "aggs": {
                    "errors": {
                      "cardinality": {
                        "field": "id"
                      }
                    }
                  }
                },
                "qps": {
                  "bucket_script": {
                    "buckets_path": {
                      "times": "times",
                      "duration": "duration"
                    },
                    "script": "(int)(params.times / params.duration * 1000000)"
                  }
                }
              }
            },
            "times": {
              "sum_bucket": {
                "buckets_path": "timeArea>times"
              }
            },
            "duration": {
              "sum_bucket": {
                "buckets_path": "timeArea>duration"
              }
            },
            "qps": {
              "stats_bucket": {
                "buckets_path": "timeArea>qps"
              }
            },
            "errors": {
              "sum_bucket": {
                "buckets_path": "timeArea>errors>errors"
              }
            }
          }
        }
      }
    }
  }
}
```

> `aggs.traces`: 根据`traceName`分桶,`traceName`相同的`span`属于入口相同的调用   
> `aggs.spans`: 根据`name`分桶,`name`相同的`span`其调用服务(`client`)和被调用服务(`server`)相同   
> `aggs.client, aggs.server`: 取出`client`和`server`属性   
> `timeArea`: 根据时间(60s)分段统计，目的是计算出峰值`qps`   
> `timeArea.qps`: `qps`的计算公式为`调用总次数/总耗时`   
> `qps.stats_bucket`: 统计`qps`的最大最小平均等   

#### 6. 参考

* [Spring Cloud Sleuth](https://github.com/spring-cloud/spring-cloud-sleuth)

* [Zipkin](https://zipkin.io/)

* [Storm](http://storm.apache.org/releases/1.2.1/index.html)

* [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

* [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)