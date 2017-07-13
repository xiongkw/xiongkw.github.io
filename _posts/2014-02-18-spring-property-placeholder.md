---
layout: post
title: Spring property-placeholder源码解读
categories: [编程, java, spring]
tags: [java, spring]
---

> spring编码中，我们常常使用`占位符${}`的方式把配置从代码(xml)中抽离到配置文件

```xml
	<context:property-placeholder location="classpath*:conf/jdbc.properties" />
	
	<bean class="com.xx.DataSource">
    		<property name="url" value="${jdbc.url}" />
    		<property name="username" value="${jdbc.username}" />
    		<property name="password" value="${jdbc.password}" />
    </bean>
```

`context`是spring-context中定义的一个命令空间，从`META-INF/spring.handlers`中找到其对应handler
```
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
```
ContextNamespaceHandler:
```java
registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
```
PropertyPlaceholderBeanDefinitionParser.getBeanClass
```java
    // As of Spring 3.1, the default value of system-properties-mode has changed from
    // 'FALLBACK' to 'ENVIRONMENT'. This latter value indicates that resolution of
    // placeholders against system properties is a function of the Environment and
    // its current set of PropertySources
    if (element.getAttribute(SYSTEM_PROPERTIES_MODE_ATTRIB).equals(SYSTEM_PROPERTIES_MODE_DEFAULT)) {
        return PropertySourcesPlaceholderConfigurer.class;
    }

    // the user has explicitly specified a value for system-properties-mode. Revert
    // to PropertyPlaceholderConfigurer to ensure backward compatibility.
    return PropertyPlaceholderConfigurer.class;
```
这里根据`system-properties-mode`配置的不同返回了不同的`bean class`，在spring3.1之前，默认返回的是`PropertyPlaceholderConfigurer`，下面先看`PropertyPlaceholderConfigurer`

PropertyPlaceholderConfigurer继承自PlaceholderConfigurerSupport，这里先看PlaceholderConfigurerSupport   
PlaceholderConfigurerSupport继承了PropertyResourceConfigurer，而PropertyResourceConfigurer实现了`BeanFactoryPostProcessor, PriorityOrdered`两个接口   
> BeanFactoryPostProcessor是BeanFactory后处理器，其处理器的调用发生在BeanFactory初始化完成之后，bean实例化之前.   
> spring中`*Processor`和`Ordered`总是成对出现，因为`Processor`可以定义多个，而谁先process谁后process就需要使用`order`来指明了.

PropertyResourceConfigurer.postProcessBeanFactory
```java
    Properties mergedProps = mergeProperties();

    // Convert the merged properties, if necessary.
    convertProperties(mergedProps);

    // Let the subclass process the properties.
    processProperties(beanFactory, mergedProps);
```

PropertiesLoaderSupport.mergeProperties
```java
protected Properties mergeProperties() throws IOException {
    Properties result = new Properties();

    if (this.localOverride) {
        // Load properties from file upfront, to let local properties override.
        loadProperties(result);
    }

    if (this.localProperties != null) {
        for (Properties localProp : this.localProperties) {
            CollectionUtils.mergePropertiesIntoMap(localProp, result);
        }
    }

    if (!this.localOverride) {
        // Load properties from file afterwards, to let those properties override.
        loadProperties(result);
    }

    return result;
}
```
两个属性：
* localProperties: 本地属性变量，可以在xml中配置为placeholder的属性
* localOverride: 本地属性是否可以被覆盖，Properties中后加载的属性可以覆盖已经存在的属性，这里实际就是一个谁先加载的问题

这里的PropertiesLoaderSupport.loadProperties也有一个先后顺序
```java
if (this.locations != null) {
    for (Resource location : this.locations) {
        if (logger.isInfoEnabled()) {
            logger.info("Loading properties file from " + location);
        }
        try {
            PropertiesLoaderUtils.fillProperties(
                    props, new EncodedResource(location, this.fileEncoding), this.propertiesPersister);
        }
        catch (IOException ex) {
            if (this.ignoreResourceNotFound) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Could not load properties from " + location + ": " + ex.getMessage());
                }
            }
            else {
                throw ex;
            }
        }
    }
}
```
> 如果两个属性文件中存在相同的key，则在locations中后定义的会覆盖先定义的   
> `ignoreResourceNotFound`的作用：是否忽略属性文件加载异常，通常我们不会忽略，因为属性文件加载异常表示程序的设计可能出了问题。

PropertiesLoaderSupport.convertProperties
```java
/**
 * Convert the given property value from the properties source to the value
 * which should be applied.
 * <p>The default implementation simply returns the original value.
 * Can be overridden in subclasses, for example to detect
 * encrypted values and decrypt them accordingly.
 * @param originalValue the original value from the properties source
 * (properties file or local "properties")
 * @return the converted value, to be used for processing
 * @see #setProperties
 * @see #setLocations
 * @see #setLocation
 * @see #convertProperty(String, String)
 */
protected String convertPropertyValue(String originalValue) {
    return originalValue;
}
```
> convertProperties的作用是提供一个转换功能，例如为了安全考虑，我们在属性文件中定义的password可能是密文，而这里则可以通过方法重载实现解密。

PropertiesLoaderSupport.processProperties最终调用了PlaceholderConfigurerSupport.doProcessProperties

PlaceholderConfigurerSupport实现了两个接口`BeanNameAware, BeanFactoryAware`
```java
    // Check that we're not parsing our own bean definition,
    // to avoid failing on unresolvable placeholders in properties file locations.
    if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
        BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
        try {
            visitor.visitBeanDefinition(bd);
        }
    }
```
> 实现这两个接口的目的：   
> 1. 不对自已处理   
> 2. 只对定义自己的beanFactory处理，即`placeholder`的作用范围是其所在的beanFactory

BeanDefinitionVisitor.visitBeanDefinition
```java
public void visitBeanDefinition(BeanDefinition beanDefinition) {
    visitParentName(beanDefinition);
    visitBeanClassName(beanDefinition);
    visitFactoryBeanName(beanDefinition);
    visitFactoryMethodName(beanDefinition);
    visitScope(beanDefinition);
    visitPropertyValues(beanDefinition.getPropertyValues());
    ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
    visitIndexedArgumentValues(cas.getIndexedArgumentValues());
    visitGenericArgumentValues(cas.getGenericArgumentValues());
}
```
> 可以看到，这里对除了`property'之外的``parent,class,scope`...都可以处理，原来我们用到的只是冰山一角

占位符处理逻辑在PropertyPlaceholderHelper.parseStringValue
```java
protected String parseStringValue(
			String strVal, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

    StringBuilder result = new StringBuilder(strVal);

    int startIndex = strVal.indexOf(this.placeholderPrefix);
    while (startIndex != -1) {
        int endIndex = findPlaceholderEndIndex(result, startIndex);
        if (endIndex != -1) {
            String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
            String originalPlaceholder = placeholder;
            if (!visitedPlaceholders.add(originalPlaceholder)) {
                throw new IllegalArgumentException(
                        "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
            }
            // Recursive invocation, parsing placeholders contained in the placeholder key.
            placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
            // Now obtain the value for the fully resolved key...
            String propVal = placeholderResolver.resolvePlaceholder(placeholder);
            if (propVal == null && this.valueSeparator != null) {
                int separatorIndex = placeholder.indexOf(this.valueSeparator);
                if (separatorIndex != -1) {
                    String actualPlaceholder = placeholder.substring(0, separatorIndex);
                    String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                    propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
                    if (propVal == null) {
                        propVal = defaultValue;
                    }
                }
            }
            if (propVal != null) {
                // Recursive invocation, parsing placeholders contained in the
                // previously resolved placeholder value.
                propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
                if (logger.isTraceEnabled()) {
                    logger.trace("Resolved placeholder '" + placeholder + "'");
                }
                startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
            }
            else if (this.ignoreUnresolvablePlaceholders) {
                // Proceed with unprocessed value.
                startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
            }
            else {
                throw new IllegalArgumentException("Could not resolve placeholder '" +
                        placeholder + "'" + " in string value \"" + strVal + "\"");
            }
            visitedPlaceholders.remove(originalPlaceholder);
        }
        else {
            startIndex = -1;
        }
    }

    return result.toString();
}
```
* 从`while (startIndex != -1)`看，一个属性是可以同时有多个占位符的，如`${name}${value}`
* 从`placeholder = parseStringValue`的递归看，占位符是支持嵌套的，如`${name${value}}`
* `if (propVal == null && this.valueSeparator != null)`是对默认值的处理

`resolvePlaceholder`发生在PropertyPlaceholderConfigurer.resolvePlaceholder
```java
protected String resolvePlaceholder(String placeholder, Properties props, int systemPropertiesMode) {
    String propVal = null;
    if (systemPropertiesMode == SYSTEM_PROPERTIES_MODE_OVERRIDE) {
        propVal = resolveSystemProperty(placeholder);
    }
    if (propVal == null) {
        propVal = resolvePlaceholder(placeholder, props);
    }
    if (propVal == null && systemPropertiesMode == SYSTEM_PROPERTIES_MODE_FALLBACK) {
        propVal = resolveSystemProperty(placeholder);
    }
    return propVal;
}
```
> 注意这里有一个`systemPropertiesMode`，其取值有三个   
> `SYSTEM_PROPERTIES_MODE_NEVER`，永远不使用系统属性(用`-Dkey=value`指定，同时`searchSystemEnvironment`决定是否使用系统环境变量)   
> `SYSTEM_PROPERTIES_MODE_FALLBACK`，默认值，使用系统属性作为后备，即属性文件中找不到时才去系统属性中查找   
> `SYSTEM_PROPERTIES_MODE_OVERRIDE`，和`FALLBACK`相反，即优先使用系统属性

本文开头在PropertyPlaceholderBeanDefinitionParser.getBeanClass中根据`system-properties-mode`配置的不同返回了不同的`bean class`，我们看另一个实现`PropertySourcesPlaceholderConfigurer`

和`PropertyPlaceholderConfigurer`相同PropertySourcesPlaceholderConfigurer也继承自`PlaceholderConfigurerSupport`，并且重写了`postProcessBeanFactory`方法
```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    if (this.propertySources == null) {
        this.propertySources = new MutablePropertySources();
        if (this.environment != null) {
            this.propertySources.addLast(
                new PropertySource<Environment>(ENVIRONMENT_PROPERTIES_PROPERTY_SOURCE_NAME, this.environment) {
                    @Override
                    public String getProperty(String key) {
                        return this.source.getProperty(key);
                    }
                }
            );
        }
        try {
            PropertySource<?> localPropertySource =
                new PropertiesPropertySource(LOCAL_PROPERTIES_PROPERTY_SOURCE_NAME, mergeProperties());
            if (this.localOverride) {
                this.propertySources.addFirst(localPropertySource);
            }
            else {
                this.propertySources.addLast(localPropertySource);
            }
        }
        catch (IOException ex) {
            throw new BeanInitializationException("Could not load properties", ex);
        }
    }

    processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
    this.appliedPropertySources = this.propertySources;
}
```
* 这里用`Environment`替换了`PropertyPlaceholderConfigurer`中的`systemProperties`和`searchSystemEnvironment`，所以可以支持`Profile`
* 使用PropertySources替换了`PropertyPlaceholderConfigurer`中的`java.util.Properties`

这里的占位符处理逻辑在`PropertySourcesPropertyResolver`，和`PropertyPlaceholderConfigurer`不同    
PropertySourcesPropertyResolver.processProperties
```java
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
			final ConfigurablePropertyResolver propertyResolver) throws BeansException {

    propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);
    propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);
    propertyResolver.setValueSeparator(this.valueSeparator);

    StringValueResolver valueResolver = new StringValueResolver() {
        @Override
        public String resolveStringValue(String strVal) {
            String resolved = ignoreUnresolvablePlaceholders ?
                    propertyResolver.resolvePlaceholders(strVal) :
                    propertyResolver.resolveRequiredPlaceholders(strVal);
            return (resolved.equals(nullValue) ? null : resolved);
        }
    };

    doProcessProperties(beanFactoryToProcess, valueResolver);
}
```
> 占位符解析逻辑同`PropertyPlaceholderConfigurer`，都是在`PropertyPlaceholderHelper.parseStringValue`

占位符替换发生在PropertySourcesPropertyResolver.getProperty
```java
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
    boolean debugEnabled = logger.isDebugEnabled();
    if (logger.isTraceEnabled()) {
        logger.trace(String.format("getProperty(\"%s\", %s)", key, targetValueType.getSimpleName()));
    }
    if (this.propertySources != null) {
        for (PropertySource<?> propertySource : this.propertySources) {
            if (debugEnabled) {
                logger.debug(String.format("Searching for key '%s' in [%s]", key, propertySource.getName()));
            }
            Object value;
            if ((value = propertySource.getProperty(key)) != null) {
                Class<?> valueType = value.getClass();
                if (resolveNestedPlaceholders && value instanceof String) {
                    value = resolveNestedPlaceholders((String) value);
                }
                if (debugEnabled) {
                    logger.debug(String.format("Found key '%s' in [%s] with type [%s] and value '%s'",
                            key, propertySource.getName(), valueType.getSimpleName(), value));
                }
                if (!this.conversionService.canConvert(valueType, targetValueType)) {
                    throw new IllegalArgumentException(String.format(
                            "Cannot convert value [%s] from source type [%s] to target type [%s]",
                            value, valueType.getSimpleName(), targetValueType.getSimpleName()));
                }
                return this.conversionService.convert(value, targetValueType);
            }
        }
    }
    if (debugEnabled) {
        logger.debug(String.format("Could not find key '%s' in any property source. Returning [null]", key));
    }
    return null;
}
```

附：   
占位符默认值的处理会发生一个有趣的现象：在定义多个(虽然不推荐，但也不可避免)`PropertyPlaceholderConfigurer`bean的时候，如果第一个`PropertyPlaceholderConfigurer`中找不到占位符对应的属性值，那这个占位符就会使用默认值。 如何解决？
* 使用`localProperties`定义默认值，设置`localOverride=true`，使其优先加载
* 使用`property-override`，见[Spring property-override源码解读]({{site.url}}/2014/02/23/spring-property-override)