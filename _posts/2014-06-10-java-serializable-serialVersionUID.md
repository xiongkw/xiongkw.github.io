---
layout: post
title: Java对象序列化中的serialVersionUID
categories: [编程, java]
tags: [序列化]
---


废话不多说，贴异常
```
local class incompatible: stream classdesc serialVersionUID = 4031403137955036735, local class serialVersionUID = -6537460252478266442

```

原因：由于反序列化时class的serialVersionUID和序列化时不一致导致

查看源码，发现类里没有定义serialVersionUID

查看java.io.Serializable文档

>  If a serializable class does not explicitly declare a serialVersionUID, then the serialization runtime will calculate a default serialVersionUID value for that class based on various aspects of the class, as described in the Java(TM) Object Serialization Specification. However, it is strongly recommended that all serializable classes explicitly declare serialVersionUID values, since the default serialVersionUID computation is highly sensitive to class details that may vary depending on compiler implementations, and can thus result in unexpected InvalidClassExceptions during deserialization. Therefore, to guarantee a consistent serialVersionUID value across different java compiler implementations, a serializable class must declare an explicit serialVersionUID value. It is also strongly advised that explicit serialVersionUID declarations use the private modifier where possible, since such declarations apply only to the immediately declaring class--serialVersionUID fields are not useful as inherited members. Array classes cannot declare an explicit serialVersionUID, so they always have the default computed value, but the requirement for matching serialVersionUID values is waived for array classes.

如果Serializable实现类中没有显示定义serialVersionUID，则编译时编译器会根据源码生成一个默认值，并且不同编译器生成的值也不同，生成算法对源码和编译环境细节非常敏感

解决方法：在Serializable实现类中定义一个确定的serialVersionUID值

这就是忽略IDE警告的后果，良好的编码习惯必须像强迫症一样追求完美