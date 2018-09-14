---
layout: post
title: 基于elk和kafka的日志收集系统
categories: [编程, java, linux]
tags: [elk, elasticsearch, logstash, kibana, kafka, filebeat]
---


> `elk`是指`elasticsearch、logstash、kibana`，本来也是一个成熟的日志收集方案，由于业务系统日志数据量比较大(`TB/天`)，所以使用`Kafka`做流量消峰

#### 1. 总体架构

```
    +-------------------------------------------------------+
    |    +-------+-------+-------+                          |
    |    |       |       |       |                          |
    |    |docker |docker |docker |        +----------+      |       +---------+          +---------+          +---------+
    |    |       |       |       |        |          |      |       |         |          |         |          |         |
    |    +-------+-------+-------+  read  | FileBeat | -----------> | Kafka   | -------> | Logstash| -------> |   ES    |
    |    |         Disk          | -----> |          |      |       |         |          |         |          |         |
    |    +-----------------------+        +----------+      |       +---------+          +----------          +---------+
    |                       Host                            |
    +-------------------------------------------------------+
```

* 1 容器中的进程把日志写入宿主机指定目录(通过挂载方式)
* 2 宿主机上的`FileBeat`进程扫描指定目录下的日志文件，读取并发送到`Kafka`
* 3 `Logstash`从`Kafka`中读取日志记录解析成`json`格式并写到`ES`
* 4 应用从`ES`中查询日志

#### 2. 通过Ansible安装FileBeat

`FileBeat`需要安装在每台宿主机上，这里通过`Ansible`批量安装，详情省略，参考[Ansible入门]({{site.url}}/2018/06/28/linux-ansible/)

新建`FileBeat`配置：test.yml

```yaml
filebeat.modules:
- module: kafka
  log:
    enabled: true
- module: osquery
  result:
    enabled: true
filebeat.prospectors:
- type: log
  enabled: true
  paths:
    - /logs/*.log
  fields:
    type: mylog
  fields_under_root: true
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after
output.kafka:
  enabled: true
  hosts: {{ kafka_addr }}
  topic: {{ kafka_topic }}
setup.template.settings:
setup.kibana:
logging.to_files: true
logging.files:

#xpack监控，`inventory_hostname`是`ansible`内置变量，表示当前主机名称(ip)
name: host_{{ inventory_hostname }}
xpack.monitoring:
  enabled: true
  elasticsearch:
    hosts: ["http://192.168.1.100:8200"]
~
```

> 这里扫描`/logs/*.log`，并发送到`Kafka`

启动命令

```
filebeat -c test.yml
```

#### 3. Logstash安装

下载`Logstash`解压，并安装`x-pack`插件

```
$ tar zxvf logstash-6.2.4.tar.gz
$ cd logstash-6.2.4
$ bin/logstash-plugin install file:///home/fool/x-pack-6.2.4.zip
```

修改`config/logstash.yml`配置

```yaml
node.name: 101
pipeline.workers: 4

#xpack监控
xpack.monitoring.elasticsearch.url: ["http://192.168.1.100:8200"]

```

编写`pipeline` mylog.conf

```
input {
    kafka {
        bootstrap_servers => "192.168.1.100:9200"
        client_id => logstash
        group_id => logstash
        topics => ["mylog"]
        codec => "json"
    }
}

filter {
   if [type] == "mylog" {
      grok {
           match => { "message" => "%{TIMESTAMP_ISO8601:logDateStr}\s+\[%{DATA:thread}\]\s+%{DATA:level}\s+"}
      }

      date {
              match => ["logDateStr", "yyyy-MM-dd HH:mm:ss,SSS", "ISO8601"]
              target => "logDate"
              locale => "cn"
      }
   }

   mutate {
       remove_field => ["offset", "@version", "prospector", "beat"]
   }

   if (![logDateStr]) {
     mutate {
       add_field => {
         "logDate" => "%{@timestamp}"
       }
     }
   }

}

output {
   elasticsearch {
       hosts => [ "192.168.1.100:8200" ]
       index => "mylog-%{+YYYY-MM-dd}"
       template => "./mylog.tpl"
       template_name => "mylog"
       template_overwrite => true
       document_type => "mylog"
   }
}

```

启动`logstash`

```
$ bin/logstash -f mylog.conf
```

#### 4. 安装ES

> 参考[ElasticSearch安装]({{site.url}}/2018/06/21/elasticsearch-install/)

安装`x-pack`插件：

```
$ tar zxvf elasticsearch-6.2.4.tar.gz
$ cd elasticsearch-6.2.4
$ bin/elasticsearch-plugin install file:///home/fool/x-pack-6.2.4.zip
```

#### 5. 安装Kafka

> 参考 [Kafka入门]({{site.url}}/2017/08/10/kafka-startup/)
> 参考 [Kafka Manager试用]({{site.url}}/2018/08/05/kafka-manager/)

#### 5. 安装Kibana

下载解压并安装`x-pack`插件

```
$ tar zxvf kibana-6.2.4.tar.gz
$ cd kibana-6.2.4
$ bin/kibana-plugin install file:///home/fool/x-pack-6.2.4.zip
```

修改`kibana.yml`

```yaml
server.port: 8080
elasticsearch.url: "http://192.168.1.100:8200"
```

#### 7. 监控

在`Kibana`中监控`ES、Logstash、FileBeat`：

![]({{site.url}}/public/2018-09-14-elk-startup_1.png)

![]({{site.url}}/public/2018-09-14-elk-startup_2.png)

在`Kafka-Manager`中监控`Kafka`队列消费情况：

![]({{site.url}}/public/2018-09-14-elk-startup_3.png)

#### 8. 参考

* [Elasticsearch Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
* [Logstash Reference](https://www.elastic.co/guide/en/logstash/6.2/index.html)
* [Kibana User Guide](https://www.elastic.co/guide/en/kibana/current/index.html)
* [Filebeat Reference](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
* [Apache Kafka](http://kafka.apache.org/)