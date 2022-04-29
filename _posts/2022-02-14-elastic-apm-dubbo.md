---
layout: post
title: Elastic APM之dubbo接入方案
categories: [编程, elastic]
tags: [apm]
---

> Elastic APM Agent官方已有实验性的Dubbo支持，但对版本有要求，参考 [Dubbo插件](https://www.elastic.co/guide/en/apm/agent/java/current/supported-technologies-details.html#supported-networking-frameworks) 

#### 1. 总体思路
通过dubbo Filter拦截器在服务消费方和调用方切入，使用Elastic apm api记录调用信息并发送到Elastic apm服务

#### 2. 实现

##### 2.1 项目结构

```
filter
interface
provider
consumer
```

##### 2.2 filter
pom
```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.8</version>
</dependency>

<dependency>
    <groupId>co.elastic.apm</groupId>
    <artifactId>apm-agent-api</artifactId>
    <version>1.28.4</version>
</dependency>
```

consumer filter
```java
@Activate(group = CommonConstants.CONSUMER)
public class ConsumerApmFilter implements Filter {

    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        Transaction transaction = ElasticApm.currentTransaction(); // 获取当前事务
        if (!transaction.isSampled()) {
            return invoker.invoke(invocation);
        }
        Span span = transaction.startSpan("external", "dubbo", null); // 开启duboo调用span
        try (final Scope ignored = span.activate()) {
            String name = "DUBBO " + invocation.getInvoker().getInterface().getName() + "#" + invocation.getMethodName();
            span.setName(name); // 设置span名称
            span.injectTraceHeaders((key, value) -> RpcContext.getContext().setAttachment(key, value)); // 通过RpcContext传递trace header信息
            return invoker.invoke(invocation);
        } catch (Exception e) {
            span.setLabel("arguments", JSON.toJSONString(invocation.getArguments())); // 调用异常时记录参数信息
            span.captureException(e);
            throw e;
        } finally {
            span.end();
        }
    }

}
```

provider filter
```java
@Activate(group = CommonConstants.PROVIDER)
public class ProviderApmFilter implements Filter {

    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        Transaction transaction = ElasticApm.startTransactionWithRemoteParent(invocation::getAttachment); // 使用服务方传递的trace header信息，开启新事务
        if (!transaction.isSampled()) {
            return invoker.invoke(invocation);
        }
        try (final Scope ignored = transaction.activate()) {
            String name = invocation.getInvoker().getInterface().getName() + "#" + invocation.getMethodName();
            transaction.setFrameworkName("Apache-Dubbo");
            transaction.setName(name); // 设置事务名称
            transaction.setType(Transaction.TYPE_REQUEST);
            return invoker.invoke(invocation);
        } catch (Exception e) {
            transaction.setLabel("arguments", JSON.toJSONString(invocation.getArguments())); // 异常时记录参数信息
            transaction.captureException(e);
            throw e;
        } finally {
            transaction.end();
        }
    }

}
```

注册filter(src\main\resources\META-INF\dubbo\com.alibaba.dubbo.rpc.Filter)
```
consumerApmFilter=com.demo.dubbo.filter.ConsumerApmFilter
providerApmFilter=com.demo.dubbo.filter.ProviderApmFilter
```

##### 2.3 interface

```java
public interface IHelloService {
    String sayHello(String name);
}
```

##### 2.4 provider

IHelloService
```
@DubboService(version = "1.0.0")
public class HelloServiceImpl implements IHelloService {
    public String sayHello(String name) {
        return "Hello: " + name;
    }
}
```

```yaml
spring:
  application:
    name: dubbo-provider
dubbo:
  scan:
    base-packages: com.demo.dubbo.service
  protocol:
    name: dubbo
    port: 20221
  registry:
    address: N/A
  consumer:
    filter: consumerApmFilter
  provider:
    filter: providerApmFilter
```

##### 2.5 consumer



在controller中调用 helloService
```
@RestController
public class HelloController {
    @DubboReference(version = "1.0.0", url = "dubbo://127.0.0.1:20221", methods = {@Method(name = "sayHello")})
    private IHelloService helloService;

    @GetMapping("/hello")
    public String hello(@RequestParam String name) {
        return helloService.sayHello(name);
    }

}
```

```yaml
server:
  port: 20222
spring:
  application:
    name: dubbo-consumer

dubbo:
  consumer:
    filter: consumerApmFilter
  provider:
    filter: providerApmFilter
```

#### 3. 测试

使用elastic-apm-agent启动provider和consumer，并访问HelloController

在kibana-APM中查看

![]({{site.url}}/public/images/2022-02-14-elastic-apm-dubbo.png)
