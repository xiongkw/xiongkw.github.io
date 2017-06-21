---
layout: post
title: spring自定义命名空间
categories: [编程, java, spring]
tags: [java, spring, namespace]
---

spring xml配置中通常为了简化配置或者实现增强功能，会引入自定义命名空间，如下以DataSource为例

### 1. spring xml中配置dataSource
目的，使用如下配置定义一个DataSource
```xml
<ds:ds id="myDataSource" name="myName" />
```

### 2. xsd定义
ds.xsd
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns="http://www.my.com/schema/my"
	xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:beans="http://www.springframework.org/schema/beans"
	targetNamespace="http://www.ds.com/schema/ds"
	elementFormDefault="qualified" attributeFormDefault="unqualified">

	<xsd:import namespace="http://www.springframework.org/schema/beans" />

	<xsd:element name="ds">
		<xsd:complexType>
			<xsd:complexContent>
				<xsd:extension base="beans:identifiedType">
					<xsd:attribute name="name" type="xsd:string" use="required" />
				</xsd:extension>
			</xsd:complexContent>
		</xsd:complexType>
	</xsd:element>


</xsd:schema>
```
spring xml中引入命名空间
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:ds="http://www.my.com/schema/ds"
	xsi:schemaLocation="
     http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.my.com/schema/ds http://www.my.com/schema/ds.xsd">
     
</beans>
```
### 3. spring schemas和handlers定义
META-INFO/spring.schemas
```properties
http\://www.ds.com/schema/ds.xsd=com/my/ds/config/ds.xsd
```
META-INFO/spring.handlers
```properties
http\://www.ds.com/schema/ds=com.my.ds.config.DataSourceNamespaceHandler
```

### 4. NamespaceHandler
```java
public class DataSourceNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		registerBeanDefinitionParser("ds", new DataSourceBeanDefinitionParser());
	}

}
```

### 5. BeanDefinitionParser
BeanDefinitionParser
```java
public class DatasourceBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {


	private static final String PROP_URL = "url";
	private static final String PROP_USERNAME = "username";
	private static final String PROP_PASSWORD = "password";

	@Override
	protected void doParse(Element element, BeanDefinitionBuilder builder) {
		String name = element.getAttribute("name");
        Properties props = getDataSourcePropertiesByName(name);
		builder.addPropertyValue("url", props.getProperty("url"));
		builder.addPropertyValue("username", props.getProperty("username"));
		builder.addPropertyValue("password", props.getProperty("password"));

	}
	
	private Properties getDataSourcePropertiesByName(String name){
	    //...
	    
	}

	@Override
	protected Class<?> getBeanClass(Element element) {
		return BasicDataSource.class;
	}

}
```
