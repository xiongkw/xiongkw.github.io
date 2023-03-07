---
layout: post
title: Kafka-Stream导出promtheus指标
categories: [java]
tags: [kafka]
---

> Kafka Stream有大量监控指标，但是默认只能通过JMX查看，好在框架提供了扩展接口

#### 1. JMX方式

开启JmxReporter：配置kafka stream的metric.reporters参数
```
metric.reporters=org.apache.kafka.common.metrics.JmxReporter
```

使用JDK自带工具jconsole查看，略...

#### 2. 导出prometheus格式

##### 2.1 添加依赖包

基于springboot actuator实现

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
	<version>2.7.8</version>
</dependency>
<dependency>
	<groupId>io.micrometer</groupId>
	<artifactId>micrometer-core</artifactId>
	<version>1.10.4</version>
</dependency>
<dependency>
	<groupId>io.micrometer</groupId>
	<artifactId>micrometer-registry-prometheus</artifactId>
	<version>1.10.4</version>
</dependency>
```

##### 2.2 实现MetricsReporter接口

```java
public class PrometheusReporter implements MetricsReporter {

    private MeterRegistry meterRegistry;

    @Override
    public void init(List<KafkaMetric> metrics) {
        meterRegistry = ApplicationUtils.getBean(MeterRegistry.class);
        if (meterRegistry == null || metrics == null || metrics.size() == 0) {
            return;
        }
        for (KafkaMetric metric : metrics) {
            register(metric);
        }
    }

    private void register(KafkaMetric metric) {
        MetricName metricName = metric.metricName();
        String name = metricName.name();

        Map<String, String> tags = metricName.tags();
        List<Tag> tagList = tags.entrySet().stream()
                .map(e -> Tag.of(e.getKey(), e.getValue()))
                .collect(Collectors.toList());
        tagList.add(instanceTag);

        meterRegistry.gauge(getMetricName(name), tagList, metric, (e) -> {
            Object val = metric.metricValue();
            if (val instanceof Measurable) {
                return ((Measurable) val).measure(metric.config(), Time.SYSTEM.milliseconds());
            }
            if (val instanceof Number) {
                return ((Number) val).doubleValue();
            }
            return 0;
        });
    }

    private String getMetricName(String name) {
        return name.replaceAll("-", "_");
    }

    @Override
    public void metricChange(KafkaMetric metric) {
        register(metric);
    }

    @Override
    public void metricRemoval(KafkaMetric metric) {
        MetricName metricName = metric.metricName();
        String name = metricName.name();
        Map<String, String> tags = metricName.tags();

        List<Tag> tagList = tags.entrySet().stream()
                .map(e -> Tag.of(e.getKey(), e.getValue()))
                .collect(Collectors.toList());

        Meter.Id id = new Meter.Id(name, Tags.of(tagList), null, null, Meter.Type.GAUGE);
        meterRegistry.remove(id);
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }

}


```

##### 2.3 配置kafka stream的metric.reporters参数

```
metric.reporters=org.example.PrometheusReporter
```

##### 2.4 springboot actuator配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
      base-path: /actuator
  metrics:
    tags:
      application: ${spring.application.name}

```

##### 2.5 查看指标

浏览器访问 http:127.0.0.1:8080/actuator/prometheus
