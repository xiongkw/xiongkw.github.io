---
layout: post
title: Spring中和Bean相关的接口
categories: [编程, java, spring]
tags: [bean]
---

> `Spring`中的`InitializingBean DisposableBean BeanPostProcessor FactoryBean`四个接口

#### 1.InitializingBean
源码：
```java
/**
 * Invoked by a BeanFactory after it has set all bean properties supplied
 * (and satisfied BeanFactoryAware and ApplicationContextAware).
 * <p>This method allows the bean instance to perform initialization only
 * possible when all bean properties have been set and to throw an
 * exception in the event of misconfiguration.
 * @throws Exception in the event of misconfiguration (such
 * as failure to set an essential property) or if initialization fails.
 */
void afterPropertiesSet() throws Exception;
```
> `bean`初始化接口，用于自定义初始化操作，调用发生在`bean`属性注入完成后    

`bean`初始化的几种方法:
* `InitializingBean`
* `@PostConstruct`
* `xml init-method`
* `@Bean(initMethod="init")`

#### 2.DisposableBean
源码：
```java
/**
 * Invoked by a BeanFactory on destruction of a singleton.
 * @throws Exception in case of shutdown errors.
 * Exceptions will get logged but not rethrown to allow
 * other beans to release their resources too.
 */
void destroy() throws Exception;
```
> bean销毁接口，用于自定义销毁操作，调用发生在实例销毁之前

`bean`销毁的几种方法:
* `DisposableBean`
* `@PreDestroy`
* `xml destroy-method`
* `@Bean(destroyMethod="destroy")`

#### 3.BeanPostProcessor
源码：
```java
/**
 * Apply this BeanPostProcessor to the given new bean instance <i>before</i> any bean
 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
 * or a custom init-method). The bean will already be populated with property values.
 */
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

/**
 * Apply this BeanPostProcessor to the given new bean instance <i>after</i> any bean
 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
 * or a custom init-method). The bean will already be populated with property values.
 */
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
```
> `bean`后处理器接口，用于自定义`bean`实例化操作，调用发生在`bean`初始化之前或之后

#### 4.FactoryBean
源码：
```java
T getObject() throws Exception;

Class<?> getObjectType();

boolean isSingleton();
```
> 工厂`bean`接口，可以完全定制`bean`的实例化过程