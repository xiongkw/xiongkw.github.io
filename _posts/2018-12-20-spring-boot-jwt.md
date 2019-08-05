---
layout: post
title: Spring-boot集成JWT
categories: [编程, java, spring]
tags: [spring-boot, jwt]
---


> 分布式架构下通常使用缓存来实现会话共享，由此会引入一个`重量级`的缓存组件，除此以外我们还可以使用轻量级的`JWT`来实现无会话的服务

#### 1. 关于JWT 

* `JWT`: JSON Web Token (JWT)，服务端给用户签发的一个记录了用户信息的`token`，用户下次请求时带上这个`token`，服务端会根据`token`解码用户身份，带来的好处是服务端不需要保存会话了，坏处是需要额外的解码计算

`JWT`解决了分布式架构下会话共享这一痛点，使得服务端无状态更容易扩展，但也存在一些问题：

* 服务端无法废止一个正常使用的`token`（虽然也可以通过一些额外逻辑实现，但肯定不如`HttpSession.invalidate()`来得痛快）
* `token`容易被截取和盗用，所以需要使用`HTTPS`协议传输（好像`sessionId`一样也有同样问题，不过`sessionId`一般是在`cookie`中，而`cookie`不可跨域）
* 在`token`中记录更多的用户信息（例如权限）会带来更大的流量和计算开销，不记录的话则服务器处理每次请求都要从数据库或缓存获取（又绕回`session`了）

所以又一次印证了一个真理（废话）：没有好与不好，只有合不合适

> 参考[JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)

#### 2. spring-boot集成JWT

添加`java-jwt`依赖
```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.4.1</version>
</dependency>
```

> `java-jwt`是一个`JWT`的`Java`实现(废话，看名字就知道了)，参考[Java-JWT](https://github.com/auth0/java-jwt)

##### 2.1 编写一个私有资源

```java
@GetMapping("/secret")
public String data(){
    return "My secret...";
}
```

> 不做权限控制的情况下是可以直接访问的

##### 2.2 编写一个认证接口

```java
@PostMapping(value = "/login")
public String login(@RequestParam String username, @RequestParam String password, HttpServletResponse response) throws IOException {
    if ("fool".equals(username) && "1234".equals(password)) {
        return getJwtToken(username);
    }
    response.sendError(400, "用户认证失败");
    return null;
}

private String getJwtToken(String username) {
    return JWT.create().withIssuer("fool").withAudience(username)
            // 在token中写入用户信息
            .withClaim("username", username)
            .withClaim("role", "admin")
            .sign(Algorithm.HMAC256("secret"));
}
```

> 用户认证通过后返回一个`token`

##### 2.3 编写token认证拦截器

```java
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new JwtInterceptor()).addPathPatterns("/secret");
    }

    class JwtInterceptor extends HandlerInterceptorAdapter {
        JWTVerifier verifier = JWT.require(Algorithm.HMAC256("secret")).withIssuer("fool").build();

        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            String token = request.getHeader("X-AUTH-TOKEN");
            try {
                DecodedJWT verify = verifier.verify(token);
                // 获取token中的用户信息
                String username = verify.getClaim("username").asString();
                String role = verify.getClaim("role").asString();
                // ...
                return true;
            } catch (JWTVerificationException e) {
                throw new RuntimeException("Unauthorized!");
            }
        }
    }
}
```

> 拦截`/secret`访问，并做`token`校验

##### 2.4 访问secret

1. 直接访问`/secret`，响应`Unauthorized!`
2. 请求`/login`获取`token`
3. 设置请求头`X-AUTH-TOKEN: $token`访问`/secret`能正常访问
4. 设置请求头`X-AUTH-TOKEN:`为一个无效的`token`后再次访问，响应`Unauthorized!`

#### 3. 参考

* [JWT](https://jwt.io/)

* [Java-JWT](https://github.com/auth0/java-jwt)

* [你在用 JWT 代替 Session?](https://blog.csdn.net/weixin_41153791/article/details/82291144)