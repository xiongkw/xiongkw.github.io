---
layout: post
title: Spring Cloud Sleuth和Zipkin
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

##### 1.4 运行
分别启动`服务提供者`和`消费者`并访问
```
http://127.0.0.1:8002/hi?name=Tom
```

进入`Zipkin Server`页面
```
http://127.0.0.1:9411
```

查看`Trace`
![]({{site.url}}/public/2018-04-09-spring-sleuth-and-zipkin-1.png)

`Consumer`详情
![]({{site.url}}/public/2018-04-09-spring-sleuth-and-zipkin-2.png)

`Producer`详情
![]({{site.url}}/public/2018-04-09-spring-sleuth-and-zipkin-3.png)

#### 2. 源码分析

##### 2.1 spring-cloud-sleuth-core

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

> 可以看到`sleuth`中定义的是默认行为，例如`DefaultSpanNamer. NoOp*`

##### 2.2 spring-cloud-sleuth-zipkin

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

##### 2.3 Sleuth如何工作

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
    		if (!(servletRequest instanceof HttpServletRequest) || !(servletResponse instanceof HttpServletResponse)) {
    			throw new ServletException("Filter just supports HTTP requests");
    		}
    		HttpServletRequest request = (HttpServletRequest) servletRequest;
    		HttpServletResponse response = (HttpServletResponse) servletResponse;
    		String uri = this.urlPathHelper.getPathWithinApplication(request);
    		boolean skip = this.skipPattern.matcher(uri).matches()
    				|| Span.SPAN_NOT_SAMPLED.equals(ServletUtils.getHeader(request, response, Span.SAMPLED_NAME));
    		Span spanFromRequest = getSpanFromAttribute(request);
    		if (spanFromRequest != null) {
    			continueSpan(request, spanFromRequest);
    		}
    		if (log.isDebugEnabled()) {
    			log.debug("Received a request to uri [" + uri + "] that should not be sampled [" + skip + "]");
    		}
    		// in case of a response with exception status a exception controller will close the span
    		if (!httpStatusSuccessful(response) && isSpanContinued(request)) {
    			Span parentSpan = parentSpan(spanFromRequest);
    			processErrorRequest(filterChain, request, new TraceHttpServletResponse(response, parentSpan), spanFromRequest);
    			return;
    		}
    		String name = HTTP_COMPONENT + ":" + uri;
    		Throwable exception = null;
    		try {
    			spanFromRequest = createSpan(request, skip, spanFromRequest, name);
    			filterChain.doFilter(request, new TraceHttpServletResponse(response, spanFromRequest));
    		} catch (Throwable e) {
    			exception = e;
    			errorParser().parseErrorTags(tracer().getCurrentSpan(), e);
    			if (log.isErrorEnabled()) {
    				log.error("Uncaught exception thrown", e);
    			}
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

> 使用`zipkin api`是需要编写代码的，`sleuth`使用`Filter`实现无侵入的调用链追踪

#### 3. 参考

* [Spring Cloud Sleuth](https://github.com/spring-cloud/spring-cloud-sleuth)

* [Zipkin](https://zipkin.io/)