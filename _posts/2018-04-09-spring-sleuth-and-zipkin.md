---
layout: post
title: Spring Cloud Sleuth和Zipkin源码分析
categories: [编程, java, spring]
tags: [spring cloud, sleuth, zipkin, trace]
---


> `Spring Cloud Sleuth`用于微服务之间的调用链追踪，以下是一个简单的例子

#### 1. 一个例子

##### 1.1 Zipkin Server
pom
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.9.RELEASE</version>
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

<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-server</artifactId>
</dependency>

<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-autoconfigure-ui</artifactId>
</dependency>
```

运行
```
java zipkin.server.ZipkinServer
```

> 查看`ZipkinServer`源码，发现其实际就是执行了`SpringApplication.run`

##### 1.2 服务提供者

pom
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

`Application.java`
```java
@SpringBootApplication
@RestController
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    public static final Logger LOGGER = LoggerFactory.getLogger(Application.class);

    @Autowired
    private RestTemplate restTemplate;

    @Bean
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }

    @GetMapping("/hi")
    public Object hi(@RequestParam String name) {
        LOGGER.info("hi: {}", name);
        return "Hi: " + name;
    }

}
```

`application.yml`

```yaml
spring:
  application:
    name: service-producer
  zipkin:
    base-url: http://127.0.0.1:9411
  sleuth:
    sampler:
      percentage: 1
server:
  port: 8001
```

> `spring.sleuth.sampler.percentage=1`: 设置采样率为`100%`

##### 1.3 服务消费者
pom
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

`Application.java`

```java
@SpringBootApplication
@RestController
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    public static final Logger  LOGGER = LoggerFactory.getLogger(Application.class);

    @Autowired
    private RestTemplate restTemplate;

    @Bean
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }

    @GetMapping("/hi")
    public Object hi(@RequestParam String name) {
        LOGGER.info("hi: {}", name);
        return restTemplate.getForObject("http://127.0.0.1:8001/hi?name="+name, String.class);
    }

}
```

`application.yml`
```yaml
spring:
  application:
    name: service-consumer
  zipkin:
    base-url: http://127.0.0.1:9411
  sleuth:
    sampler:
      percentage: 1
server:
  port: 8002
```

#### 2. 运行

##### 2.1 启动服务
分别启动`服务提供者`和`消费者`并访问
```
http://127.0.0.1:8002/hi?name=Tom
```

进入`Zipkin Server`页面
```
http://127.0.0.1:9411
```
##### 2.2 查看Trace
查看`Trace`
![]({{site.url}}/public/images/2018-04-09-spring-sleuth-and-zipkin-1.png)

##### 2.3 查看消费者调用信息
`Consumer`详情
![]({{site.url}}/public/images/2018-04-09-spring-sleuth-and-zipkin-2.png)

##### 2.4 查看提供者调用信息
`Producer`详情
![]({{site.url}}/public/images/2018-04-09-spring-sleuth-and-zipkin-3.png)

#### 3. 源码分析

##### 3.1 spring-cloud-sleuth-core

`TraceAutoConfiguration`

```java
@Bean
@ConditionalOnMissingBean
public Random randomForSpanIds() {
    return new Random();
}

@Bean
@ConditionalOnMissingBean
public Sampler defaultTraceSampler() {
    return NeverSampler.INSTANCE;
}

@Bean
@ConditionalOnMissingBean(Tracer.class)
public Tracer sleuthTracer(Sampler sampler, Random random,
        SpanNamer spanNamer, SpanLogger spanLogger,
        SpanReporter spanReporter, TraceKeys traceKeys) {
    return new DefaultTracer(sampler, random, spanNamer, spanLogger,
            spanReporter, this.properties.isTraceId128(), traceKeys);
}

@Bean
@ConditionalOnMissingBean
public SpanNamer spanNamer() {
    return new DefaultSpanNamer();
}

@Bean
@ConditionalOnMissingBean
public SpanReporter defaultSpanReporter() {
    return new NoOpSpanReporter();
}

@Bean
@ConditionalOnMissingBean
public SpanAdjuster defaultSpanAdjuster() {
    return new NoOpSpanAdjuster();
}

```

`Sleuth`的重要组件:

* `Random`: 用于生成随机`span id`
* `Sampler`: 采样器
* `Tracer`: `trace`记录器
* `SpanReporter`: `span`报告器
* `SpanAdjuster`: `span`校验器
* `ErrorParser`: 异常解析器

> 可以看到`sleuth`中定义的是默认行为，例如`DefaultSpanNamer. NoOp*`

##### 3.2 spring-cloud-sleuth-zipkin

`ZipkinAutoConfiguration`

```java
@Bean
@ConditionalOnMissingBean
public ZipkinRestTemplateCustomizer zipkinRestTemplateCustomizer(ZipkinProperties zipkinProperties) {
    return new DefaultZipkinRestTemplateCustomizer(zipkinProperties);
}

@Configuration
@ConditionalOnClass(RefreshScope.class)
protected static class RefreshScopedPercentageBasedSamplerConfiguration {
    @Bean
    @RefreshScope
    @ConditionalOnMissingBean
    public Sampler defaultTraceSampler(SamplerProperties config) {
        return new PercentageBasedSampler(config);
    }
}

@Configuration
@ConditionalOnMissingClass("org.springframework.cloud.context.config.annotation.RefreshScope")
protected static class NonRefreshScopePercentageBasedSamplerConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public Sampler defaultTraceSampler(SamplerProperties config) {
        return new PercentageBasedSampler(config);
    }
}
	
@Bean
public SpanReporter zipkinSpanListener(Reporter<Span> reporter, EndpointLocator endpointLocator,
        Environment environment) {
    return new ZipkinSpanReporter(reporter, endpointLocator, environment, this.spanAdjusters);
}
```

> 这里是`zipkin`实现，所以`sleuth`和`zipkin`之间是`接口`与`实现`的关系

##### 3.3 Sleuth api

`sleuth`提供了多种方式来记录调用链，如下
```
org.springframework.cloud.sleuth.instrument
                                            |-async
                                            |-hystrix
                                            |-messaging
                                            |-rxjava
                                            |-scheduling
                                            |-web
                                            |-zuul
```

这里先看最简单的一种`web.TraceFilter`
```java
@Order(TraceFilter.ORDER)
public class TraceFilter extends GenericFilterBean {
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
    			FilterChain filterChain) throws IOException, ServletException {
    		try {
    			spanFromRequest = createSpan(request, skip, spanFromRequest, name);
    			filterChain.doFilter(request, new TraceHttpServletResponse(response, spanFromRequest));
    		} catch (Throwable e) {
    			throw e;
    		} finally {
    			if (isAsyncStarted(request) || request.isAsyncStarted()) {
    				if (log.isDebugEnabled()) {
    					log.debug("The span " + spanFromRequest + " will get detached by a HandleInterceptor");
    				}
    				// TODO: how to deal with response annotations and async?
    				return;
    			}
    			detachOrCloseSpans(request, response, spanFromRequest, exception);
    		}
    	}
}
```

* `createSpan`中会尝试从`request`中获取`parent span`
* `span`的`report`动作发生在`detachOrCloseSpans`
* 从`TODO`看这里不支持异步请求

> 使用`zipkin api`是需要编写代码的，`sleuth`使用`Filter`实现无侵入的调用链追踪

##### 3.4 业务日志如何和调用链关联

`TraceEnvironmentPostProcessor`
```java
public void postProcessEnvironment(ConfigurableEnvironment environment,
        SpringApplication application) {
    Map<String, Object> map = new HashMap<String, Object>();
    // This doesn't work with all logging systems but it's a useful default so you see
    // traces in logs without having to configure it.
    if (Boolean.parseBoolean(environment.getProperty("spring.sleuth.enabled", "true"))) {
        map.put("logging.pattern.level",
                "%5p [${spring.zipkin.service.name:${spring.application.name:-}},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]");
    }
    // TODO: Remove this in 2.0.x. For compatibility we always set to true
    if (!environment.containsProperty(SPRING_AOP_PROXY_TARGET_CLASS)) {
        map.put(SPRING_AOP_PROXY_TARGET_CLASS, "true");
    }
    addOrReplace(environment.getPropertySources(), map);
}
```

> 这里定义`logging.pattern.level`为`%5p [${spring.zipkin.service.name:${spring.application.name:-}},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]`   
> `%X{xx}`是使用`MDC(Mapped Diagnostic Context)`线程变量
 
那么`X-B3-TraceId`和`X-B3-SpanId`是从哪来的呢？

`DefaultTracer.createSpan`
```java
public Span createSpan(String name, Sampler sampler) {
    String shortenedName = SpanNameUtil.shorten(name);
    Span span;
    if (isTracing()) {
        span = createChild(getCurrentSpan(), shortenedName);
    }
    else {
        long id = createId();
        span = Span.builder().name(shortenedName)
                .traceIdHigh(this.traceId128 ? createTraceIdHigh() : 0L)
                .traceId(id)
                .spanId(id).build();
        if (sampler == null) {
            sampler = this.defaultSampler;
        }
        span = sampledSpan(span, sampler);
        this.spanLogger.logStartedSpan(null, span);
    }
    return continueSpan(span);
}
```
 
`Slf4jSpanLogger.logStartedSpan`
```java
public void logStartedSpan(Span parent, Span span) {
    MDC.put(Span.SPAN_ID_NAME, Span.idToHex(span.getSpanId()));
    MDC.put(Span.SPAN_EXPORT_NAME, String.valueOf(span.isExportable()));
    MDC.put(Span.TRACE_ID_NAME, span.traceIdString());
    log("Starting span: {}", span);
    if (parent != null) {
        log("With parent: {}", parent);
        MDC.put(Span.PARENT_ID_NAME, Span.idToHex(parent.getSpanId()));
    }
}
```

> 这里把`X-B3-TraceId`和`X-B3-SpanId`写入到了`MDC`中

#### 4. 参考

* [Spring Cloud Sleuth](https://github.com/spring-cloud/spring-cloud-sleuth)

* [Zipkin](https://zipkin.io/)