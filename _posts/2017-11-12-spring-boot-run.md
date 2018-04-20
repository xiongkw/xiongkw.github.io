---
layout: post
title: spring-boot启动过程源码分析
categories: [编程, java, spring]
tags: [spring-boot]
---


> `spring-boot`中的一些重要接口和注解

#### 1. 工程搭建
`pom`

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.9.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

`Application`

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

调试运行，进入SpringApplication.run

#### 2. SpringApplication.initialize

```java
private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    this.webEnvironment = deduceWebEnvironment();
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

> `Application.class`作为一个`source`

##### 2.1 WebEnvironment

```java
private boolean deduceWebEnvironment() {
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return false;
        }
    }
    return true;
}
```

> 通过判断类路径中是否存在`org.springframework.web.context.ConfigurableWebApplicationContext`和`javax.servlet.Servlet`来决定是否使用`web环境`

##### 2.2 ApplicationContextInitializer

> 从`META-INF/spring.factories`中获取`ApplicationContextInitializer`实例   
> `SpringFactoriesLoader.loadFactoryNames(type, classLoader)`: 获取`facroty`的`api`

##### 2.3 ApplicationListener

> 从`META-INF/spring.factories`中获取`ApplicationListener`实例

##### 2.4 mainApplicationClass
````java
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
````

#### 3. SpringApplication.run

```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

> `StopWatch`: `spring`提供的计时器工具，下次不用写`System.currentTimeMillis`了

##### 3.1 SpringApplicationRunListener

这里是通过`spring.factories`获取`SpringApplicationRunListener`
```java
public interface SpringApplicationRunListener {

	/**
	 * Called immediately when the run method has first started. Can be used for very
	 * early initialization.
	 */
	void starting();

	/**
	 * Called once the environment has been prepared, but before the
	 * {@link ApplicationContext} has been created.
	 * @param environment the environment
	 */
	void environmentPrepared(ConfigurableEnvironment environment);

	/**
	 * Called once the {@link ApplicationContext} has been created and prepared, but
	 * before sources have been loaded.
	 * @param context the application context
	 */
	void contextPrepared(ConfigurableApplicationContext context);

	/**
	 * Called once the application context has been loaded but before it has been
	 * refreshed.
	 * @param context the application context
	 */
	void contextLoaded(ConfigurableApplicationContext context);

	/**
	 * Called immediately before the run method finishes.
	 * @param context the application context or null if a failure occurred before the
	 * context was created
	 * @param exception any run exception or null if run completed successfully.
	 */
	void finished(ConfigurableApplicationContext context, Throwable exception);

}
```

> `SpringApplicationRunListener`的作用是监听`ApplicationContext`的生命周期   
> 看其默认配置是`EventPublishingRunListener`，用于发出不同的`ApplicationEvent`，例如`ApplicationStartedEvent,ApplicationEnvironmentPreparedEvent,ApplicationPreparedEvent,ApplicationReadyEvent,ApplicationFailedEvent`，可以配合`ApplicationListener`来监听并处理这些事件

##### 3.2 prepareEnvironment

根据不同环境创建不同的`Environment`
```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    if (this.webEnvironment) {
        return new StandardServletEnvironment();
    }
    return new StandardEnvironment();
}
```

```java
protected void configureEnvironment(ConfigurableEnvironment environment,
        String[] args) {
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}
```

> 处理`propertySource`和`Profile`

```java
protected void configurePropertySources(ConfigurableEnvironment environment,
			String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(
                new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(new SimpleCommandLinePropertySource(
                    name + "-" + args.hashCode(), args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}
```

> 命令行中的参数也会被解析到`PropertySource`


##### 3.3 createApplicationContext

```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            contextClass = Class.forName(this.webEnvironment
                    ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}
```

> 根据不同环境创建不同`ApplicationContext`

##### 3.4 prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // Add boot specific singleton beans
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    // Load the sources
    Set<Object> sources = getSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[sources.size()]));
    listeners.contextLoaded(context);
}
```

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

> 调用`ApplicationContextInitializer.initialize`，这里可以自定义初始化

```java
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug(
                "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    BeanDefinitionLoader loader = createBeanDefinitionLoader(
            getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}
```

> 加载`source`配置中的`bean`到`ApplicationContext`

##### 3.5 refreshContext

```java
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext) applicationContext).refresh();
}
```

> 调用`applicationContext.refresh`

##### 3.6 afterRefresh

```java
protected void afterRefresh(ConfigurableApplicationContext context,
        ApplicationArguments args) {
    callRunners(context, args);
}

private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<Object>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```

> 这里是提供一个后置处理的入口(见`*PostProcessor`)，`ApplicationRunner`和`CommandLineRunner`功能相同，唯一区别是参数形式不一样