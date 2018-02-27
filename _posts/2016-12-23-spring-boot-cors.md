---
layout: post
title: spring-boot中开启cors
categories: [编程, java, spring]
tags: [spring-boot, cors]
---


> `spring-boot`中开启`cors`   

#### 1. 一个例子

```java
@Configuration
public class MvcConfig extends WebMvcConfigurerAdapter {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/greeting").allowedOrigins("*");
        registry.addMapping("/hello").allowedOrigins("http://hello.com");
    }
    
}
```

> 设置`*`(所有域)都能访问`/greeting`   
> 设置`http://hello.com`可以访问`/hello`

#### 2. 参考

* [HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
* [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)
* [jsonp和cors]({{site.url}}/2016/12/15/jsonp-cors/)