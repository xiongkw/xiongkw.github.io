---
layout: post
title: Spring中和Bean相关的接口
categories: [编程, java, spring]
tags: [spring, bean]
---

> Spring中的`InitializingBean DisposableBean BeanPostProcessor FactoryBean`四个接口

1.InitializingBean
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
> bean初始化接口，用于自定义初始化操作，调用发生在bean属性注入完成后    

bean初始化的三种方法:
* InitializingBean
* @PostConstruct
* init-method

2.DisposableBean
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

bean销毁的三种方法:
* DisposableBean
* @PreDestroy
* destroy-method

3.BeanPostProcessor
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
> bean后处理器接口，用于自定义bean实例化操作，调用发生在bean初始化之前或之后

4.FactoryBean
```java
    T getObject() throws Exception;

	Class<?> getObjectType();

	boolean isSingleton();
```
> 工厂bean接口，可以完全定制bean的实例化过程