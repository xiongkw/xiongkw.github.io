---
layout: post
title: Eureka入门
categories: [编程, java]
tags: [eureka]
---

> 本文演示如何部署Eureka服务端，并使用spring-cloud注册和发现服务实例

#### 1. Eureka服务端

> Eureka不提供服务端安装文件，这里使用spring boot方式搭建一个Eureka服务端

##### 1.1 加入spring-boot工程pom依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

##### 1.2 编写spring-boot启动类：
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

##### 1.3 属性配置：
```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  server:
    #开户自我保护时，服务掉线后不会被Eureka服务端删除
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 4000
  client:
    # 这里作为单纯的Eureka服务端，不注册自己
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://localhost:8762/eureka/
```

#### 2. Eureka客户端

##### 2.1 pom依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

##### 2.2 属性配置
```yaml
spring:
  application:
    name: myservice
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

##### 2.3 获取调用url
```java
@Autowired
private EurekaClient discoveryClient;

public String serviceUrl() {
    InstanceInfo instance = discoveryClient.getNextServerFromEureka("MYSERVICE", false);
    return instance.getHomePageUrl();
}

```

> Eureka中服务名总是转换为大写

#### 3. 从Eureka中读取服务实例
> 非spring系统使用原生Eureka api读取服务实例

##### 3.1 pom依赖
pom.xml
```xml
<dependency>
    <groupId>com.netflix.eureka</groupId>
    <artifactId>eureka-client</artifactId>
    <version>1.6.2</version>
</dependency>
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```

##### 3.2 java代码
java
```java
@PostConstruct
public void init(){
    MyDataCenterInstanceConfig instanceConfig = new MyDataCenterInstanceConfig();
    InstanceInfo instanceInfo = new EurekaConfigBasedInstanceInfoProvider(instanceConfig).get();
    ApplicationInfoManager applicationInfoManager = new ApplicationInfoManager(instanceConfig, instanceInfo);
    AbstractDiscoveryClientOptionalArgs args = new DiscoveryClient.DiscoveryClientOptionalArgs();
    args.setEventListeners(Sets.<EurekaEventListener>newHashSet(this));
    discoveryClient = new DiscoveryClient(applicationInfoManager, new DefaultEurekaClientConfig(), args);
}

public void onEvent(EurekaEvent event) {
    if (discoveryClient == null) {
        return;
    }
    Applications applications = discoveryClient.getApplications();
    List<Application> list = applications.getRegisteredApplications();
    for(Application app: list){
        List<InstanceInfo> instances = app.getInstances();
        for (InstanceInfo info : instances) {
            String ipAddr = info.getIPAddr();
            int port = info.getPort();
            // ...
        }
    }
    
}
```

##### 3.3 属性配置
eureka-client.properties
```properties
eureka.registration.enabled=false

## configuration related to reaching the eureka servers
eureka.preferSameZone=true
eureka.shouldUseDns=false
eureka.serviceUrl.default=http://localhost:8761/eureka/
client.refresh.interval=10

eureka.decoderName=JacksonJson
```
> 原生api中属性文件名称和路径不可自定义


#### 4. 几个问题

* 1.客户端发现服务的时间延迟比较长，原因是Eureka客户端从服务端读取服务实例的时间间隔默认为30s，可通过修改参数`client.refresh.interval`控制
* 2.服务断线后，Eureka服务端却还长时间存在，原因是Eureka客户端心跳的时间间隔默认30s，服务端认为其超时的时间为90s，同时服务端清除断线客户端的频率默认为60s。可通过修改参数`eviction-interval-timer-in-ms`调整清除超时客户端的频率，修改参数`eureka.instance.leaseExpirationDurationInSeconds`调整服务失效时间，修改参数`eureka.instance.leaseRenewalIntervalInSeconds`调整客户端发送心跳的频率
* 3.客户端120s后才能感知服务的断线，原因是客户端对服务实例的缓存时间为30s，加上2中90s和60s，最多可达180s