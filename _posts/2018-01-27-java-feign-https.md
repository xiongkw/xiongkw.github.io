---
layout: post
title: Java访问https
categories: [编程, java, spring]
tags: [feign, https]
---

> 使用自签名证书搭建https服务，通过浏览器访问时，浏览器会提示证书不可信任，但是仍可`继续浏览此网站(不推荐)`

#### 1. https服务

部署并启动https服务，参见[Spring-boot搭建https服务]({{ site.url}}/2018/01/26/spring-boot-https/)

#### 2. 客户端编写
使用feign访问https服务

```java
public class Client {
    interface MyService{
        @RequestLine("POST /hello")
        String hello();
    }

    public static void main(String[] args) {
        MyService myService = Feign.builder().target(MyService.class, "https://www.myapp.com:8443");
        myService.hello();
    }

}
```

#### 3. 访问

直接运行，异常了
```
Exception in thread "main" feign.RetryableException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target executing POST https://www.myapp.com:8443/hello
	at feign.FeignException.errorExecuting(FeignException.java:67)
	at feign.SynchronousMethodHandler.executeAndDecode(SynchronousMethodHandler.java:104)
	at feign.SynchronousMethodHandler.invoke(SynchronousMethodHandler.java:76)
	at feign.ReflectiveFeign$FeignInvocationHandler.invoke(ReflectiveFeign.java:103)
	at com.aep.ecloud.oauthweb.$Proxy2.token(Unknown Source)
	at com.aep.ecloud.oauthweb.Main.main(Main.java:19)
```
> 原因是找不到对应的证书

#### 4. 导入证书

先导出
```
keytool -exportcert -alias myapp -keystore myapp.keystore -file myapp.cer -rfc
```

再导入
```
keytool -importcert -file myapp.cer -keystore $JAVA_HOME\jre\lib\security\cacerts -alias myapp
```

再次运行正常了