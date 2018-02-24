---
layout: post
title: Spring中的Aware接口
categories: [编程, java, spring]
tags: [aware]
---


> `spring`中有很多`aware`接口，比如`ApplicationContextAware`，即能够感知`ApplicationContext`的，何谓能够感知？通俗讲就是能够获取到对`ApplicationContext`的引用。

#### 1. 一个例子
一个`bean`如何完成`感知`呢?
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

#### 2. 源码查看

`感知`在何时发生？这里截取了`ApplicationContextAwareProcessor`的两段代码
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

> `ApplicationContextAwareProcessor`的一个`BeanPostProcessor`，感知的动作在`postProcessBeforeInitialization`，即`bean`初始化(比如`InitializingBean`)之前完成

#### 3. 常见Aware接口

* `ApplicationContextAware`: 上下文感知的
* `ApplicationEventPublisherAware`: 事件发布器感知
* `BeanClassLoaderAware`: 类加载器感知
* `BeanFactoryAware`:  `bean factory`感知
* `BeanNameAware`: `bean id`感知
* `EnvironmentAware`: 环境感知
* `ResourceLoaderAware`: 资源加载器感知
* `ServletConfigAware`: `servlet config`感知
* `ServletContextAware`: `servlet context`感知