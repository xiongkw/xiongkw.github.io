---
layout: post
title: Springboot推送指标到Prometheus Pushgateway
categories: [java]
tags: [prometheus]
---

> Prometheus一般采用主动scrape的方式从exporter获取指标，在防火墙、NAT、容器环境，或者应用根本就不开端口的情况就要用push方式了

#### 1. 安装Pushgateway

在Pushgateway[github仓库](https://github.com/prometheus/pushgateway/tags)下载

运行

```
$ ./pushgateway --web.listen-address=:9091
```

#### 2. 添加依赖包

以springboot actuator为例

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

<dependency>
	<groupId>io.prometheus</groupId>
	<artifactId>simpleclient_pushgateway</artifactId>
	<version>0.16.0</version>
</dependency>

```

#### 3. 配置

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
    export:
      prometheus:
        push-gateway:
          enabled: true
          job: streams
          baseUrl: "http://127.0.0.1:9091"
```

只需要配置`management.metrics.export.prometheus.push-gateway`即可，actuator已经帮我们自动配好了，参考`org.springframework.boot.actuate.autoconfigure.metrics.export.prometheus.PrometheusMetricsExportAutoConfiguration`

#### 4. 其它

##### 4.1 应用挂了

> Pushgateway有个问题就是应用挂了以后，exporter接口还是能查到之前上报的指标

可设置参数`management.metrics.export.prometheus.push-gateway.shutdown-operation=DELETE`结合优雅kill解决，只是不适用暴力kill（-9）

##### 4.2 用户认证

开启Pushgateway的Basic Auth认证，参考[TLS and basic authentication](https://github.com/prometheus/pushgateway)

```
$ ./pushgateway --web.listen-address=:9091 --web.config.file=web-config.yml
```

web-config.yml内容，参考[Web configuration](https://github.com/prometheus/exporter-toolkit/blob/master/docs/web-configuration.md)

```yaml
basic_auth_users:
   abc: **************************
```

password加密方法(依赖httpd-tools包)
```
$ yum install httpd-tools
$ htpasswd -nBC 10 "" | tr -d ':\n'
```

应用配置：
```
management:
  metrics:
    export:
      prometheus:
        push-gateway:
          username: abc
          password: 123456
```

#### 5. 参考

* [WHEN TO USE THE PUSHGATEWAY](https://prometheus.io/docs/practices/pushing/)
* [PUSHING METRICS](https://prometheus.io/docs/instrumenting/pushing/)
* [Prometheus Pushgateway](https://github.com/prometheus/pushgateway)
