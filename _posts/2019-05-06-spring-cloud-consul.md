---
layout: post
title: spring-cloud-consul试用
categories: [编程, java, spring]
tags: [consul]
---


> 新版`spring-cloud`逐步从`Netflix`迁移出来，例如`spring-cloud-consul`

#### 1. consul简介

> `Consul` 是一套开源的分布式服务发现和配置管理系统，由 `HashiCorp` 公司用 `Go` 语言开发

##### 1.1 功能

* 服务注册发现：基本功能
* 健康检查：通过健康检查来维护服务的可用状态
* 配置管理：意味着一套系统同时提供`注册中心`和`配置中心`功能

##### 1.2 主流服务注册中心对比

| | euerka | Consul | zookeeper | etcd |
| ---- | ------ | ------ | ------ | -------- |
| 服务健康检查 | 可配支持 | 服务状态，内存，硬盘等	(弱)长连接，keepalive | 连接心跳 |
| 多数据中心 | — | 支持 | — | — |
| kv 存储服务 | — | 支持 | 支持 | 支持 |
| 一致性 | — | raft | paxos | raft|
| cap | ap | cp | cp | cp |
| 使用接口(多语言能力) | http（sidecar） | 支持 http 和 dns	客户端 | http/grpc |
| watch 支持 | 支持 long polling/大部分增量 | 全量/支持long polling | 支持 | 支持 long polling |
| 自身监控 | metrics | metrics | — | metrics |
| 安全 | — | acl/https | acl | https 支持（弱） |
| spring cloud 集成 | 已支持 | 已支持 | 已支持 | 已支持 |


#### 2. 安装

[下载](https://www.consul.io/downloads.html)，解压

```
$ mkdir consul_1.4.4
$ unzip consul_1.4.4_linux_amd64.zip -d consul_1.4.4
$ cd consul_1.4.4
$ ./consul -v
Consul v1.4.4
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
```

#### 3. 运行

```
$ nohup ./consul agent -dev -bind 192.168.1.100 -client 192.168.1.100 >std.log 2>&1 &
```

> `-bind`参数指定节点绑定的ip，`-client`参数指定节点提供服务的地址，默认都是`127.0.0.1`

#### 4. 集群成员查看

```
$ ./consul members
Node  Address         Status  Type    Build  Protocol  DC   Segment
bss3  127.0.0.1:8301  alive   server  1.4.4  2         dc1  <all>
```

#### 5. spring-cloud开发

使用[Spring Initializr](https://start.spring.io/)生成demo

##### 5.1 producer

pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>
```

bootstrap.yml

```yaml
spring:
  profiles:
    active: dev
  application:
    name: producer
  cloud:
    consul:
      host: 192.168.1.100
      port: 8500
      discovery:
        health-check-path: /actuator/health
        prefer-ip-address: true
      config:
        format: yaml
        prefix: config
        default-context: producer
        data-key: data
        profile-separator: ','
server:
  port: 8088
```

Application.java
```java
@SpringBootApplication
@EnableDiscoveryClient
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

ProducerController
```java
@RestController
@RequestMapping("/users")
public class ProducerController {

    @GetMapping("/{id}")
    public User getUserById(@PathVariable int id){
        return new User(id, "Tom");
    }
}
```

##### 5.2 consumer

bootstrap.yml

```yaml
spring:
  application:
    name: consumer
  cloud:
    consul:
      host: 192.168.1.100
      port: 8500
      discovery:
        health-check-path: /actuator/health
        prefer-ip-address: true
      config:
        format: yaml
        prefix: config
        default-context: consumer
        data-key: data
        profile-separator: ','
server:
  port: 8080
```

Application.java
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

IProducerClient.java
```java
@FeignClient("producer")
public interface IDataClient {

    @GetMapping("/users/{id}")
    User getById(@PathVariable(name = "id") int id);
}

```

ConsumerController.java

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @Autowired
    private IDataClient dataClient;

    @GetMapping("/{id}")
    public User getUserById(@PathVariable int id){
        return dataClient.getById(id);
    }

}
```

#### 6. 参考

* [Consul Documentation](https://www.consul.io/docs/index.html)
* [Spring Initializr](https://start.spring.io/)