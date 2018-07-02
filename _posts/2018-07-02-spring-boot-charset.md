---
layout: post
title: Spring-boot中的乱码
categories: [编程, java, spring]
tags: [spring-boot, charset, language]
---


> `spring-boot`中默认`http`编码为`UTF-8`，但是偶尔还是会出现中文乱码

#### 1. 现象

某个网页在`Chrome`上打开正常，但在`Safari`上打开就出现乱码

#### 2. 查看协议

查看`Chrome`和`Safari`访问时的请求和响应发现`Accept-Language`和`Content-Type`不同

| 浏览器 | Accept-Language | Content-Type |
| :--- | :--- | :--- |
| Chrome | zh-CN,zh;q=0.9 | text/html;charset=UTF-8 |
| Safari | en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7 | text/html;charset=ISO-8859-1 |

#### 3. 查看spring-boot编码

> `spring-boot`编码由`spring.http.encoding.charset`参数指定，默认是`UTF-8`，但是对不同`Accept-Language`响应不同`charset`也是正确的行为，所以问题还是出在`Safari`请求头上

这里提供一个办法，强制`spring-boot`使用`UTF-8`编码: 修改`application.yml`

```yaml
spring:
    http:
        encoding:
            force: true
```