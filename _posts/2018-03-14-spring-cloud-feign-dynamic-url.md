---
layout: post
title: spring cloud中动态url的feign client用法
categories: [编程, java, spring]
tags: [spring-cloud, feign, dynamic]
---


> `spring-cloud`中使用`feign`来调用`http`接口

#### 1. 背景

现有一多租户系统，每个租户有其不同的服务`A`(接口协议都相同，只是访问`url`不同)

#### 2. 使用@FeignClient

`spring-cloud`提供了注解`@FeignClient`可以直接使用

```java
@FeignClient("192.168.1.100:8080")
public interface ServiceA {
    
    @GetMapping("/hello")
    String sayHello(@RequestParam("name") String name);
    
}
```

查看`FeignClientFactoryBean#getObject`源码
```java
public Object getObject() throws Exception {
    FeignContext context = applicationContext.getBean(FeignContext.class);
    Feign.Builder builder = feign(context);

    if (!StringUtils.hasText(this.url)) {
        String url;
        if (!this.name.startsWith("http")) {
            url = "http://" + this.name;
        }
        else {
            url = this.name;
        }
        url += cleanPath();
        return loadBalance(builder, context, new HardCodedTarget<>(this.type,
                this.name, url));
    }
    if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
        this.url = "http://" + this.url;
    }
    String url = this.url + cleanPath();
    Client client = getOptional(context, Client.class);
    if (client != null) {
        if (client instanceof LoadBalancerFeignClient) {
            // not lod balancing because we have a url,
            // but ribbon is on the classpath, so unwrap
            client = ((LoadBalancerFeignClient)client).getDelegate();
        }
        builder.client(client);
    }
    Targeter targeter = get(context, Targeter.class);
    return targeter.target(this, builder, context, new HardCodedTarget<>(
            this.type, this.name, url));
}
```

> 发现这里的`url`来自注解定义，无法做到动态化

#### 3. 动态url改造

`ServiceA.java`

```java
public interface ServiceA {
    
    @Requestline("GET /hello?name={name}")
    String sayHello(@Param("name") String name);
    
}
```

`MyClient.java`

```java
public class MyClient {
    private ServiceA serviceA;

    @Autowired
    private FeignContext context;

    @Value("${feign.Logger.Level:BASIC}")
    private feign.Logger.Level level;

    private ThreadLocal<String> threadLocalUrl = new ThreadLocal<>();

    @PostConstruct
    public void initDcosService() {
        serviceA = Feign.builder()
                // 使用slf4j打印日志
                .logger(new Slf4jLogger(MyClient.class))
                .logLevel(level)
                // 使用spring-cloud提供的encoder和decoder
                .encoder(context.getInstance("feignEncoder", Encoder.class))
                .decoder(context.getInstance("feignDecoder", Decoder.class))
                // 实现Target接口，实现动态url功能
                .target(new Target<ServiceA>() {
                    @Override
                    public Class<ServiceA> type() {
                        return ServiceA.class;
                    }

                    @Override
                    public String name() {
                        return null;
                    }

                    @Override
                    public String url() {
                        return null;
                    }

                    @Override
                    public Request apply(RequestTemplate input) {
                        // 从线程本地变量获取当前租户的url
                        String url = threadLocalUrl.get();
                        input.insert(0, url);
                        return input.request();
                    }
                });
    }
    
    public void sayHello(String name, String tenant){
        // 根据租户信息获取其对应服务url
        String url = getUrlByTenant(tenant);
        // 把租户url写入线程本地变量
        threadLocalUrl.set(url);
        try {
            // 调用服务
            serviceA.sayHello(name);
        }finally {
            threadLocalUrl.remove();
        }
    }
    
}
```

#### 5. 参考

* [Open feign](https://github.com/OpenFeign/feign)
* [Spring-Cloud中使用Feign结合Hystrix实现微服务调用]({{site.url}}/2018/01/12/spring-cloud-feign)