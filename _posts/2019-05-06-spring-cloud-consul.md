---
layout: post
title: spring-cloud-consul试用
categories: [编程, java, spring]
tags: [consul]
---


> 新版`spring-cloud`逐步脱离`Netflix`生态，转而推崇其它产品，例如`gateway、consul`

#### 1. consul简介

> `Consul` 是一套开源的分布式服务发现和配置管理系统，`HashiCorp`公司出品，基于`Go`语言开发

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
$ nohup ./consul agent -server -bootstrap-expect 1 -ui -data-dir /data/consul -bind 192.168.1.100 -client 192.168.1.100 >std.log 2>&1 &
```

* `-server`: 以`server`方式启动
* `-bootstrap-expect`: 指定最少节点数`1`
* `-ui`：启动`ui`
* `-data`：指定数据持久化目录，否则数据只会保存在内存中，重启后会丢失
* `-bind`：指定节点绑定的`ip`
* `-client`：指定节点提供服务的地址，默认都是`127.0.0.1`

#### 4. 集群成员查看

```
$ ./consul members
Node  Address         Status  Type    Build  Protocol  DC   Segment
bss3  192.168.1.100:8301  alive   server  1.4.4  2         dc1  <all>
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
@RefreshScope
public class DataController {

    @Value("${user.name}")
    private String name;
    
    @GetMapping("/{id}")
    public User getUserById(@PathVariable int id){
        return new User(id, this.name);
    }
}
```

##### 5.2 consumer
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

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

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

##### 5.3 测试配置管理

登录`consul-ui`，创建`Key/Value`
```
Key: config/producer,dev/data
Value: 
user.name: Tom
```

运行`producer`，访问`http://127.0.0.1:8088/users/1`
```
{"id":1,"name":"Tom"}
```

##### 5.4 测试配置刷新

修改配置
```
user.name: Tomcat
```

刷新页面`http://127.0.0.1:8088/users/1`

```
{"id":1,"name":"Tomcat"}
```

> 动态刷新配置属性需要使用注解`@RefreshScope`

##### 5.5 测试服务注册与发现

运行`producer`和`consumer`，打开`consul-ui Services`

| Service | Health Checks | Tags |
| ------- | ------------- | ---- |
| consul | 1 | |
| consumer | 2 | secure=false |
| producer | 2 | secure=false |

说明服务`producer`和`consumer`都注册成功，并且健康检查通过

访问`http://127.0.0.1:8080/users/1`
```
{"id":1,"name":"Tomcat"}
```

> 使用`Feign`实现服务调用

#### 6. 参考

* [Consul Documentation](https://www.consul.io/docs/index.html)
* [Spring Initializr](https://start.spring.io/)