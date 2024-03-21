---
layout: post
title: spring-boot-actuator自定义Endpoint
categories: [编程, java, spring]
tags: []
---

> Spring Boot Actuator提供了生产级别的特性，比如健康检查、审计、指标收集、HTTP 跟踪等，帮助我们监控和管理Spring Boot应用

#### 1. 一个例子

不停机更改采样率

##### 1.1 引入actuator依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

##### 1.2 编写Endpoint

```java
@Component
@Endpoint(id = "sampler")
public class sampler {
    private int rate;

    @ReadOperation
    public int getRate() {
        return rate;
    }

    @WriteOperation
    public void setRate(int rate) {
        this.rate = rate;
    }

    public void sample(){
        // TODO
    }
}
```

##### 1.3 management配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
      base-path: /actuator
```

> 开启Endpoint访问

##### 1.4 测试

```
# 查询初始采样率
$ curl http://127.0.0.1:8080/actuator/sampler
0
# 设置采样率为30%
$ curl -X POST http://127.0.0.1:8080/actuator/sampler \
-H 'Content-Type: application/json' \
-d '{
    "rate":30
}'
# 查询采样率
$ curl http://127.0.0.1:8080/actuator/sampler
30
```

#### 2. 和Controller的区别

以上例子其实用webmvc的Controller也能实现，那两种有什么区别呢？

1. Controller偏向业务功能，而actuator则用于应用自身的监控管理
2. Controller方式只能用于web应用，而actuator可以用于非web应用
3. Controller方式只提供Http访问，而actuator可提供Http和JMX方式

#### 3. 参考

* [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.enabling)
