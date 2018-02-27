---
layout: post
title: Spring中的Processor接口
categories: [编程, java, spring]
tags: [bean]
---

> `Spring`有很多`*Processor`接口，比如`BeanFactoryPostProcessor`，提供扩展`BeanFactory`的功能

#### 1. BeanDefinitionRegistryPostProcessor
```java
/**
 * Modify the application context's internal bean definition registry after its
 * standard initialization. All regular bean definitions will have been loaded,
 * but no beans will have been instantiated yet. This allows for adding further
 * bean definitions before the next post-processing phase kicks in.
 * @param registry the bean definition registry used by the application context
 * @throws org.springframework.beans.BeansException in case of errors
 */
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
```
> 提供修改`BeanDefinitionRegistry`的入口，调用发生在`BeanDefinitionRegistry`初始化之后，`BeanFactory`初始化之前，一般用于在运行时添加或修改`bean`定义。

#### 2. BeanFactoryPostProcessor
```java
/**
 * Modify the application context's internal bean factory after its standard
 * initialization. All bean definitions will have been loaded, but no beans
 * will have been instantiated yet. This allows for overriding or adding
 * properties even to eager-initializing beans.
 * @param beanFactory the bean factory used by the application context
 * @throws org.springframework.beans.BeansException in case of errors
 */
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

```
> 提供修改`BeanFactory`的入口，调用发生在`BeanDefinition`加载之后，`bean`实例化之前，一般用于修改`bean`的属性。例如`property-placeholder和property-override`都是其实现类。

#### 3. InstantiationAwareBeanPostProcessor
```java
/**
 * Apply this BeanPostProcessor <i>before the target bean gets instantiated</i>.
 * The returned bean object may be a proxy to use instead of the target bean,
 * effectively suppressing default instantiation of the target bean.
 * <p>If a non-null object is returned by this method, the bean creation process
 * will be short-circuited. The only further processing applied is the
 * {@link #postProcessAfterInitialization} callback from the configured
 * {@link BeanPostProcessor BeanPostProcessors}.
 * <p>This callback will only be applied to bean definitions with a bean class.
 * In particular, it will not be applied to beans with a "factory-method".
 * <p>Post-processors may implement the extended
 * {@link SmartInstantiationAwareBeanPostProcessor} interface in order
 * to predict the type of the bean object that they are going to return here.
 * @param beanClass the class of the bean to be instantiated
 * @param beanName the name of the bean
 * @return the bean object to expose instead of a default instance of the target bean,
 * or {@code null} to proceed with default instantiation
 * @throws org.springframework.beans.BeansException in case of errors
 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#hasBeanClass
 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getFactoryMethodName
 */
Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

/**
 * Perform operations after the bean has been instantiated, via a constructor or factory method,
 * but before Spring property population (from explicit properties or autowiring) occurs.
 * <p>This is the ideal callback for performing field injection on the given bean instance.
 * See Spring's own {@link org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor}
 * for a typical example.
 * @param bean the bean instance created, with properties not having been set yet
 * @param beanName the name of the bean
 * @return {@code true} if properties should be set on the bean; {@code false}
 * if property population should be skipped. Normal implementations should return {@code true}.
 * Returning {@code false} will also prevent any subsequent InstantiationAwareBeanPostProcessor
 * instances being invoked on this bean instance.
 * @throws org.springframework.beans.BeansException in case of errors
 */
boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

/**
 * Post-process the given property values before the factory applies them
 * to the given bean. Allows for checking whether all dependencies have been
 * satisfied, for example based on a "Required" annotation on bean property setters.
 * <p>Also allows for replacing the property values to apply, typically through
 * creating a new MutablePropertyValues instance based on the original PropertyValues,
 * adding or removing specific values.
 * @param pvs the property values that the factory is about to apply (never {@code null})
 * @param pds the relevant property descriptors for the target bean (with ignored
 * dependency types - which the factory handles specifically - already filtered out)
 * @param bean the bean instance created, but whose properties have not yet been set
 * @param beanName the name of the bean
 * @return the actual property values to apply to to the given bean
 * (can be the passed-in PropertyValues instance), or {@code null}
 * to skip property population
 * @throws org.springframework.beans.BeansException in case of errors
 * @see org.springframework.beans.MutablePropertyValues
 */
PropertyValues postProcessPropertyValues(
        PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
        throws BeansException;
```
> 提供自定义`bean`实例化的功能

#### 4. BeanPostProcessor
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
> `bean`后处理器接口，用于自定义`bean`初始化操作，调用发生在`bean`实例化之后，初始化前后。