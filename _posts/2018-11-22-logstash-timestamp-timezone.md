---
layout: post
title: Logstash中timestamp时区的问题
categories: [编程, java]
tags: [logstash, elasticsearch, timestamp, timezone]
---

> 使用`Logstash`收集日志并存储到`ElasticSearch`，根据`timestamp`写到不同的`ES`索引(以日期区分)

#### 1. 问题

每天`8:00`以前的日志都写入了前一天的索引中

`Logstash pipeline`
```
input {
    #...
}

filter {
    #...
}

output {
    elasticsearch {
        index => "log-%{+YYYY-MM-dd}"
    }
}

```

> `ES`的索引日期定义为`%{+YYYY-MM-dd}`，即以当前`event`的`@timestamp`

#### 2. 原因

`event`的`@timestamp`为`UTC 0`时区，转换成北京时间应该`+08:00`

#### 3. 解决办法

> 查看`Logstash`文档，发现`%{+YYYY-MM-dd}`只能用于`@timestamp`字段，也没有提供格式化其它日期字段的方法

这里提供三种方法

##### 3.1 使用`log`时间覆盖`@timestamp`字段

```
filter {
    date {
        match => ["timeStr", "yyyy-MM-dd HH:mm:ss.S", "ISO8601"]
        target => "@timestamp"
        timezone => "Etc/UTC"
    }
}
```

> 缺点是丢失了`event`的`@timestamp`字段，可用在不要求`event`时间戳的场景

##### 3.2 使用log日期作为索引名

```
filter {
    dissect {
        mapping => {
            "timeStr" => "%{dateStr} %{}"
        }
    }
    
    mutate {
        gsub => ["dateStr", "-", "."]
    }
}

output {
    elasticsearch {
        index => "log-%{dateStr}"
    }
}

```

> 使用`log`日期作为索引名日期，推荐使用

##### 3.3 使用ruby脚本格式化日期

```
filter {
    ruby {
        code => "event.set('dateStr', event.get('@timestamp').time.localtime.strftime('%Y.%m.%d'))"
    }
}

output {
    elasticsearch {
        index => "log-%{dateStr}"
    }
}
```

> 使用`ruby`脚本格式化日期字段，最灵活

#### 4. 参考

* [Accessing Event Data and Fields in the Configuration](https://www.elastic.co/guide/en/logstash/current/event-dependent-configuration.html)
* [Elasticsearch output plugin](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html)
* [Date filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-date.html)
* [Ruby filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-ruby.html)