---
layout: post
title: Spring-cloud-feign源码解析
categories: [编程, java, spring]
tags: [spring-cloud, feign, ribbon, hystrix]
---


>

#### 1. Feign client的定义

##### 1.1 EnableFeignClients
从入口`EnableFeignClients`出发

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {

}
```

> `@Import(FeignClientsRegistrar.class)`表示从`FeignClientsRegistrar`导入配置
> `@Import`能导入的配置有`@Configuration classes, ImportSelector and ImportBeanDefinitionRegistrar implementations`三种

##### 1.2 FeignClientsRegistrar
`FeignClientsRegistrar`实现了`ImportBeanDefinitionRegistrar`接口

```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    // 注册一个名为default.${application.className}的bean
    registerDefaultConfiguration(metadata, registry);
    registerFeignClients(metadata, registry);
}
```

```java
public void registerFeignClients(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    // ...
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
            FeignClient.class);
    final Class<?>[] clients = attrs == null ? null
            : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        // 扫描所有带有@FeignClient注解的类
        scanner.addIncludeFilter(annotationTypeFilter);
        basePackages = getBasePackages(metadata);
    }
    // ...
}
```

`getBasePackages`的默认行为，即注解了`EnableFeignClients`的类所在路径
```java
protected Set<String> getBasePackages(AnnotationMetadata importingClassMetadata) {
    // ...
    if (basePackages.isEmpty()) {
        basePackages.add(
                ClassUtils.getPackageName(importingClassMetadata.getClassName()));
    }
    return basePackages;
}
```

`registerFeignClient`的逻辑

```java
public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // ...
    String name = getClientName(attributes);
    registerClientConfiguration(registry, name, attributes.get("configuration"));

    registerFeignClient(registry, annotationMetadata, attributes);
    // ...
}

private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    // 注册了一个名为${client.name}.FeignClientSpecification的bean
    registry.registerBeanDefinition(
            name + "." + FeignClientSpecification.class.getSimpleName(),
            builder.getBeanDefinition());
}

private void registerFeignClient(BeanDefinitionRegistry registry,
        AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();

    BeanDefinitionBuilder definition = BeanDefinitionBuilder
            .genericBeanDefinition(FeignClientFactoryBean.class);
    validate(attributes);
    definition.addPropertyValue("url", getUrl(attributes));
    definition.addPropertyValue("path", getPath(attributes));
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    definition.addPropertyValue("type", className);
    definition.addPropertyValue("decode404", attributes.get("decode404"));
    definition.addPropertyValue("fallback", attributes.get("fallback"));
    definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

    String alias = name + "FeignClient";
    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

    boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

    beanDefinition.setPrimary(primary);

    String qualifier = getQualifier(attributes);
    if (StringUtils.hasText(qualifier)) {
        alias = qualifier;
    }

    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
            new String[] { alias });
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

> 真正的`client`是使用`FeignClientFactoryBean`定义的

#### 2. FeignClientFactoryBean

`FeignClientFactoryBean`实现了`FactoryBean`接口

```java
public Object getObject() throws Exception {
    FeignContext context = applicationContext.getBean(FeignContext.class);
    // 这里的Builder实现为`HystrixFeign.Builder`，其定义在`FeignClientsConfiguration`
    Feign.Builder builder = feign(context);

    // 如果没有定义url属性，则使用服务发现
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
    // 如果有定义url则直接使用该确定的url
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

protected void configureUsingProperties(FeignClientProperties.FeignClientConfiguration config, Feign.Builder builder) {
    if (config.getLoggerLevel() != null) {
        builder.logLevel(config.getLoggerLevel());
    }
    // 这里是设置http连接超时和read超时的时间，即来源于FeignClientConfiguration
    if (config.getConnectTimeout() != null && config.getReadTimeout() != null) {
        builder.options(new Request.Options(config.getConnectTimeout(), config.getReadTimeout()));
    }
}
```

> `FeignContext`继承自`NamedContextFactory`，其使用`FeignClientsConfiguration`作为默认的配置，`FeignContext`的作用是为每一个不同名称的`FeignClient`创建一个隔离环境的`ApplicationContext`

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
        HardCodedTarget<T> target) {
    Client client = getOptional(context, Client.class);
    if (client != null) {
        builder.client(client);
        Targeter targeter = get(context, Targeter.class);
        return targeter.target(this, builder, context, target);
    }

    throw new IllegalStateException(
            "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```
> 这里得到`Client`的实现为`LoadBalancerFeignClient`，其定义在`DefaultFeignLoadBalancedConfiguration`
> `Targeter`实现为`HystrixTargeter`，其定义在`HystrixFeignTargeterConfiguration`

```java
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
						Target.HardCodedTarget<T> target) {
    if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
        return feign.target(target);
    }
    feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
    SetterFactory setterFactory = getOptional(factory.getName(), context,
        SetterFactory.class);
    if (setterFactory != null) {
        builder.setterFactory(setterFactory);
    }
    Class<?> fallback = factory.getFallback();
    if (fallback != void.class) {
        return targetWithFallback(factory.getName(), context, target, builder, fallback);
    }
    Class<?> fallbackFactory = factory.getFallbackFactory();
    if (fallbackFactory != void.class) {
        return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
    }

    return feign.target(target);
}

private <T> T targetWithFallback(String feignClientName, FeignContext context,
                                 Target.HardCodedTarget<T> target,
                                 HystrixFeign.Builder builder, Class<?> fallback) {
    T fallbackInstance = getFromContext("fallback", feignClientName, context, fallback, target.type());
    return builder.target(target, fallbackInstance);
}

public <T> T target(Target<T> target, T fallback) {
  return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null)
      .newInstance(target);
}
```

这里生成了一个代理对象
```java
public <T> T newInstance(Target<T> target) {
Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

for (Method method : target.type().getMethods()) {
  if (method.getDeclaringClass() == Object.class) {
    continue;
  } else if(Util.isDefault(method)) {
    DefaultMethodHandler handler = new DefaultMethodHandler(method);
    defaultMethodHandlers.add(handler);
    methodToHandler.put(method, handler);
  } else {
    methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
  }
}
InvocationHandler handler = factory.create(target, methodToHandler);
T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
  defaultMethodHandler.bindTo(proxy);
}
return proxy;
}
```

#### 3. 调用过程

`HystrixInvocationHandler.invoke`
```java
public Object invoke(final Object proxy, final Method method, final Object[] args)
  throws Throwable {
// early exit if the invoked method is from java.lang.Object
// code is the same as ReflectiveFeign.FeignInvocationHandler
if ("equals".equals(method.getName())) {
  try {
    Object otherHandler =
        args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
    return equals(otherHandler);
  } catch (IllegalArgumentException e) {
    return false;
  }
} else if ("hashCode".equals(method.getName())) {
  return hashCode();
} else if ("toString".equals(method.getName())) {
  return toString();
}

HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>(setterMethodMap.get(method)) {
  @Override
  protected Object run() throws Exception {
    try {
      // 真正的请求发生在这里
      return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
    } catch (Exception e) {
      throw e;
    } catch (Throwable t) {
      throw (Error) t;
    }
  }
}
return hystrixCommand.execute();
}

public Object invoke(Object[] argv) throws Throwable {
RequestTemplate template = buildTemplateFromArgs.create(argv);
Retryer retryer = this.retryer.clone();
while (true) {
  try {
    return executeAndDecode(template);
  } catch (RetryableException e) {
  // feign重试机制的实现
    retryer.continueOrPropagate(e);
    if (logLevel != Logger.Level.NONE) {
      logger.logRetry(metadata.configKey(), logLevel);
    }
    continue;
  }
}
}

```

> `HystrixInvocationHandler`使用了`RxJava`实现

```java
public Response execute(Request request, Request.Options options) throws IOException {
    try {
        URI asUri = URI.create(request.url());
        String clientName = asUri.getHost();
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
                this.delegate, request, uriWithoutHost);

        IClientConfig requestConfig = getClientConfig(options, clientName);
        return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
                requestConfig).toResponse();
    }
}
```

> 和`feign`相同，`robbin`也使用了`NamedContextFactory`，其实现为`SpringClientFactory`，默认配置为`RibbonClientConfiguration`
> 不同的是其`context`默认为懒加载，所以第一次调用会创建`context`从而花费较多时间，可通过配置为饥饿加载来解决

#### 4. 几个重要Configuration

* `FeignAutoConfiguration`:
* `FeignClientsConfiguration`
* `FeignRibbonClientAutoConfiguration`
* `RibbonClientConfiguration`

#### 5. 关于timeout

通常调用中会出现超时的情况，一般有`feign`和`hystrix`两种超时情况

##### 5.1 feign

```yaml
feign:
    client:
        config:
            default:
                connectTimeout: 10000
                readTimeout: 30000 //默认2s
```

> 参见`FeignClientConfiguration`

##### 5.2 hystrix

```yaml
hystrix:
    command:
        default:
            execution:
                timeout:
                    enabled: false
                isolation:
                    thread:
                        timeoutInMilliseconds: 30000 //默认1s
```

> 参见`HystrixCommandProperties`

##### 5.3 ribbon超时

```java
public class RibbonClientConfiguration {

	public static final int DEFAULT_CONNECT_TIMEOUT = 1000;
	public static final int DEFAULT_READ_TIMEOUT = 1000;

	@Bean
	@ConditionalOnMissingBean
	public IClientConfig ribbonClientConfig() {
		DefaultClientConfigImpl config = new DefaultClientConfigImpl();
		config.loadProperties(this.name);
		config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
		config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
		return config;
	}
}
```

> `ribbon`超时时间默认为1s



```java
public RibbonResponse execute(final RibbonRequest request, IClientConfig configOverride)
        throws IOException {
    final Request.Options options;
    if (configOverride != null) {
        options = new Request.Options(
                configOverride.get(CommonClientConfigKey.ConnectTimeout,
                        this.connectTimeout),
                (configOverride.get(CommonClientConfigKey.ReadTimeout,
                        this.readTimeout)));
    }
    else {
        options = new Request.Options(this.connectTimeout, this.readTimeout);
    }
    // ...
}
```

