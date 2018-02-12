---
layout: post
title: Spring-Cloud中使用Feign结合Hystrix实现微服务调用
categories: [编程, java, spring]
tags: [spring-cloud, Feign, Hystrix]
---

#### 1. 简介

`Feign`: Feign is a java to http client binder inspired by Retrofit, JAXRS-2.0, and WebSocket. Feign's first goal was reducing the complexity of binding Denominator uniformly to http apis regardless of restfulness.

`Hystrix`: Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems, services and 3rd party libraries, stop cascading failure and enable resilience in complex distributed systems where failure is inevitable.

#### 2. 服务提供者

##### 2.1 pom.xml
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.9.RELEASE</version>
    <relativePath />
</parent>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Edgware.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>
</dependencies>
```

##### 2.2 Application.java
```java
@SpringBootApplication
@EnableDiscoveryClient
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }

}
```

##### 2.4 application.yml
```yaml
server:
    port: 8081

spring:
    application:
        name: hello-service
    jackson:
      default-property-inclusion: non_null
    cloud:
        zookeeper:
            connect-string: 127.0.0.1:2181
```

> 使用`zookeeper`作为服务注册中心

##### 2.4 服务(Controller)

```java
@RestController
public class MyController {

    @PostMapping("/hello")
    public String hello(@RequestParam String name) {
        return "Hello: "+name;
    }
    
}
```

#### 3. 服务消费者

##### 3.1 pom.xml
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Edgware.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-feign</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>
</dependencies>
```

##### 3.2 Application.java
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }

}
```

##### 3.3 
```yaml
server:
  port: 8080

spring:
  application:
    name: hello-client
  cloud:
    zookeeper:
      connect-string: 127.0.0.1
feign:
  hystrix:
    enabled: true
```

##### 3.4 Feign服务

`IRemoteHelloService.java`
```java
@FeignClient(value = "hello-service", fallback = RemoteHelloServiceHystrixImpl.class)
public interface IRemoteHelloService {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    String hello(@RequestParam(name = "name") String name);

}
```

`RemoteHelloServiceHystrixImpl.java`
```java
@Component
public class RemoteHelloServiceHystrixImpl implements IRemoteHelloService{
    String hello(String name){
        return "Hystrix hello: " + name;
    }
}
```

##### 3.5 HelloController.java

```java
@RestController
public class HelloController{
    @Autowired
    private IRemoteHelloService remoteHelloService;

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    String hello(@RequestParam String name){
        return remoteHelloService.hello(name);
    }
    
}
```

疑问：`@FeignClient`会生成一个`IRemoteHelloService`实现`bean`，而`Hystrix`实现又是一个`IRemoteHelloService`实现，为什么`HelloController`中`@Autowired`不会报错？

查看`@FeignClient`源码，发现有一个`primary`属性：
```java
	/**
	 * Whether to mark the feign proxy as a primary bean. Defaults to true.
	 */
	boolean primary() default true;
```


#### 4. 参考文档
[Spring Cloud](http://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html)

[OpenFeign](https://github.com/OpenFeign/feign)

[Hystrix](https://github.com/Netflix/hystrix)