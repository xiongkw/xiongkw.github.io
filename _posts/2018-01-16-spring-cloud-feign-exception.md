---
layout: post
title: Spring-Cloud微服务调用中对http请求异常的处理
categories: [编程, java, spring]
tags: [spring-cloud, Feign, Hystrix]
---

#### 1. 问题
`spring-cloud`微服务中，接口一般设计成`restful`风格，其响应状态码也遵循http标准，例如

```
200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
204 NO CONTENT - [DELETE]：用户删除数据成功。
400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功。
```

通常情况下非2xx的响应会被直接屏蔽，但是一些特殊的提示我们也希望能够响应给用户，例如用444来定义用户名格式错误的异常

```java
// 如果传入的name格式错误，则响应444
User getUserByName(String name);
```

那么如何在服务调用方识别这种异常，并提示给用户(您的用户名格式不对，正确的格式应该是...)

#### 2. 使用Feign ErrorDecoder

##### 2.1 实现自定义的ErrorDecoder

```java
public class SimpleErrorDecoder extends ErrorDecoder.Default {

    @Override
    public Exception decode(String methodKey, Response response) {
        int status = response.status();
        if (status == 444) {
            throw new MyException("用户名格式错误");
        }
        return super.decode(methodKey, response);
    }
}
```

> `spring-mvc`会捕获到`MyException`并友好的提示用户

##### 2.2 配置全局范围的ErrorDecoder

```yaml
feign:
  client:
    config:
      feignName:
        errorDecoder: com.example.SimpleErrorDecoder
```

> 全局范围的`ErrorDecoder`会应用到所有的`Feign client`

##### 2.3 配置client范围的ErrorDecoder
```java
@FeignClient(value = "zuul-server", configuration = FeignConfiguration.class)
public interface IUserService {

}
```

> `client`范围的`ErrorDecoder`只对指定`client`有效

FeignConfiguration.java
```java
public class FeignConfiguration {

    @Bean
    public ErrorDecoder errorDecoder(){
        return new SimpleErrorDecoder();    
    }

}
```

##### 2.4 注意
以上方法建立在不起用`Hystrix`的情况下

```yaml
feign:
  hystrix:
    enabled: false
```

#### 3. 使用Hystrix

使用`Hystrix`则比较简单，直接在`spring-mvc`异常处理`ControllerAdvice`中捕获`HystrixRuntimeException`就可以了

```java
@ExceptionHandler(HystrixRuntimeException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public Object onHystrixRuntimeException(HystrixRuntimeException e){
    Throwable cause = e.getCause();
    if(cause instanceof FeignException){
        FeignException fe = (FeignException) cause;
        int status = fe.status();
        if(status == 444){
            return new Response(444, "用户名格式错误");
        }
    }
    return unkownError();
}
```

#### 4. 参考文档
[Spring Cloud](http://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html)