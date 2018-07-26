---
layout: post
title: Jackson序列化时忽略指定属性
categories: [编程, java]
tags: [jackson, json]
---


> 使用`Jackson`序列化`java`对象为`json`时，如何指定忽略对象的某个属性

#### 1. 使用JsonIgnore

```java
@JsonIgnore
private byte[] content;
```

> 类似的注解还有`JsonIgnoreProperties,JsonIgnoreType`，缺点是序列化和反序列化时都忽略掉了

#### 2. 使用JsonProperty.Access

```java
@JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
private byte[] content;
```

> 可以控制到只序列化(`READ_ONLY`)或者只反序列化(`WRITE_ONLY`)

#### 3. 使用JsonSerialize

```java
@JsonSerialize(using = NullSerializer.class)
private byte[] content;
```

> 貌似只能做到自定义`value`的序列化，`key`还是无法忽略


#### 4. 反序列化时如何忽略多余的字段

使用`jackson ObjectMapper`把`json`字符串反序列化为`java`对象时，如果`json`中有多余的字段，则会报错

```
Unrecognized field, not marked as ignorable
```

##### 4.1 使用JsonIgnoreProperties注解

```java
@JsonIgnoreProperties(ignoreUnknown = true)
```

##### 4.2 使用ObjectMapper配置

```java
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,false);
```

#### 5. 参考

* [Jackson Annotations](https://github.com/FasterXML/jackson-annotations/wiki/Jackson-Annotations)

* [Jackson Data-Binding Annotations](https://github.com/FasterXML/jackson-databind/wiki/Databind-Annotations)