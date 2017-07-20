---
layout: post
title: Spring中的Aware接口
categories: [编程, java, spring]
tags: [spring, aware]
---

> `spring`中有很多`aware`接口，比如`ApplicationContextAware`，即能够感知`ApplicationContext`的，何谓能够感知？通俗讲就是能够获取到对`ApplicationContext`的引用。

一个bean如何完成`感知`呢?
```java
public class AppContext implements ApplicationContextAware {
	private ApplicationContext applicationContext;

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}
}
```
> `ApplicationContextAware`有一个接口方法`setApplicationContext`，这里便可以获取到`ApplicationContext`的引用了

`感知`在何时发生？这里截取了ApplicationContextAwareProcessor的两段代码
```java
class ApplicationContextAwareProcessor implements BeanPostProcessor{
    
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
    		AccessControlContext acc = null;
    		//...
            invokeAwareInterfaces(bean);
            //...
        return bean;
    }
    	
    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof Aware) {
            //...
            if (bean instanceof ApplicationContextAware) {
                ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
            }
        }
    }
}
```
> ApplicationContextAwareProcessor的一个`BeanPostProcessor`，感知的动作在`postProcessBeforeInitialization`，即bean初始化(比如InitializingBean)之前完成

spring中常用的Aware接口：
* ApplicationContextAware 上下文感知的
* ApplicationEventPublisherAware 事件发布器感知
* BeanClassLoaderAware 类加载器感知
* BeanFactoryAware  bean factory感知
* BeanNameAware bean id感知
* EnvironmentAware 环境感知
* ResourceLoaderAware 资源加载器感知
* ServletConfigAware servlet config
* ServletContextAware servlet context