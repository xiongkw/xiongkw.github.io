---
layout: post
title: Spring property-override源码解读
categories: [编程, java, spring]
tags: [property, override]
---

> `spring`编码中，我们常常使用`override`的方式改写第三方`jar`包中`bean`的属性值

#### 1. 一个例子
```xml
<context:property-override location="classpath*:conf/dataSource.properties" />

<bean id="dataSource" class="com.xx.DataSource">
        <property name="url" value="url" />
        <property name="username" value="xxx" />
        <property name="password" value="123" />
</bean>
```

#### 2. context命令空间

`context`是`spring-context`中定义的一个命令空间，从`META-INF/spring.handlers`中找到其对应`handler`

```
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
```

`ContextNamespaceHandler:`

```java
registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
```

`PropertyOverrideBeanDefinitionParser.getBeanClass`

```java
protected Class<?> getBeanClass(Element element) {
    return PropertyOverrideConfigurer.class;
}
```

#### 3. PropertyOverrideConfigurer.processProperties

`PropertyOverrideConfigurer`继承自`PropertyResourceConfigurer`，并重写了`processProperties`方法

```java
protected void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
			throws BeansException {

    for (Enumeration<?> names = props.propertyNames(); names.hasMoreElements();) {
        String key = (String) names.nextElement();
        try {
            processKey(beanFactory, key, props.getProperty(key));
        }
        catch (BeansException ex) {
            String msg = "Could not process key '" + key + "' in PropertyOverrideConfigurer";
            if (!this.ignoreInvalidKeys) {
                throw new BeanInitializationException(msg, ex);
            }
            if (logger.isDebugEnabled()) {
                logger.debug(msg, ex);
            }
        }
    }
}
```

#### 4. PropertyOverrideConfigurer.processKey

```java
protected void processKey(ConfigurableListableBeanFactory factory, String key, String value)
        throws BeansException {

    int separatorIndex = key.indexOf(this.beanNameSeparator);
    if (separatorIndex == -1) {
        throw new BeanInitializationException("Invalid key '" + key +
                "': expected 'beanName" + this.beanNameSeparator + "property'");
    }
    String beanName = key.substring(0, separatorIndex);
    String beanProperty = key.substring(separatorIndex+1);
    this.beanNames.add(beanName);
    applyPropertyValue(factory, beanName, beanProperty, value);
    if (logger.isDebugEnabled()) {
        logger.debug("Property '" + key + "' set to value [" + value + "]");
    }
}
```

> 这里可以看出属性文件的`key`必须是`beanName.property`，并且不支持多级

#### 5. PropertyOverrideConfigurer.applyPropertyValue

`bean`属性替换的逻辑在`PropertyOverrideConfigurer.applyPropertyValue`

```java
protected void applyPropertyValue(
        ConfigurableListableBeanFactory factory, String beanName, String property, String value) {

    BeanDefinition bd = factory.getBeanDefinition(beanName);
    while (bd.getOriginatingBeanDefinition() != null) {
        bd = bd.getOriginatingBeanDefinition();
    }
    PropertyValue pv = new PropertyValue(property, value);
    pv.setOptional(this.ignoreInvalidKeys);
    bd.getPropertyValues().addPropertyValue(pv);
}
```

#### 6. 总结

对比一下`placeholder`和`override`的区别，虽然两者没什么可比性(一个是占位、一个是重写)，但他们的应用却有重叠：

* `placeholder`是占位，即一个符号，需要用具体的属性值来替换。`override`是重写，只是提供一个在属性文件中重写`bean`默认属性值的方法。
* `placeholder`需要用`<property name="username" value="${jdbc.username:root}"/>`定义，而`override`直接给定一个默认值`<property name="username" value="root"/>`
* `placeholder`可以用于`beanName,class,property,scope,property`等，而`override`只能用于`property`
* `placeholder`的属性`key`和占位符相同，而`override`的属性`key`是`bean.property`