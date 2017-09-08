---
layout: post
title: spring中classpath资源的加载和通配
categories: [编程, java, spring]
tags: [classpath, resource]
---

> Spring资源使用`AntPathMatcher`作为匹配器

1. 无通配符，完全匹配
```
classpath:conf/applicationContext.xml
```
> 只匹配`conf/applicationContext.xml`

2. 匹配一个字符
 ```
classpath:conf/applicationContext?.xml
 ```
> 匹配`conf/applicationContext1.xml`、`conf/applicationContext2.xml`等

3. 匹配多个字符
 ```
classpath:conf/applicationContext*.xml
 ```
> 匹配`conf/applicationContext1.xml`、`conf/applicationContext-abc.xml`等

4. 单目录匹配
```
classpath:*/applicationContext.xml
```
> 匹配`conf/applicationContext.xml`,`aa/applicationContext.xml`

5. 多级目录匹配
```
classpath:conf/**/applicationContext.xml
```
> 匹配`conf/applicationContext.xml`,`conf/xxx/abc/applicationContext.xml`

`classpath*:conf/applicationContext.xml`和`classpath:conf/applicationContext.xml`的区别：   
在有多个同路径资源时，`classpath*`会加载所有资源，而`classpath:`只加载找到的第一个资源
