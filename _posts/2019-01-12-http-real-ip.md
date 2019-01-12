---
layout: post
title: Java中获取HTTP请求的Client端真实IP
categories: [编程, web, java]
tags: [X-Real-IP, X-Forwarded-For]
---


> 记一次获取`HTTP`请求客户端真实`IP`方法的改进

#### 1. 最直白的方式()servlet api)

```java
public void doGet(HttpServletRequest req, HttpServletResponse res){
    String clientIp = req.getRemoteAddr();
}
```

> 以上代码在没有代理的情况下获取到的`IP`是正确的

#### 2. 通过nginx反向代理的情况

修改`nginx`配置:

nginx.conf
```
location / {
    #...
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

#### 3. X-Real-IP

通过读取请求头`X-Real-IP`获取

```java
String clientIp = request.getHeader("X-Real-IP");
```

#### 4. X-Forwarded-For

通过读取请求头`X-Forwarded-For`获取

```java
String forwardedFor = request.getHeader("X-Forwarded-For");
String clientIp = forwardedFor;
if(forwardedFor.indexOf(",") > 0){
    clientIp = forwardedFor.substring(0, forwardedFor.indexOf(","));
}
```

> `X-Forwarded-For`会保留所有经过的代理服务器`IP`地址，用`,`分隔

#### 5. 合并多种情况后的方法

```java
public static String getRealIpAddr(HttpServletRequest request) {
    String forwardedFor = request.getHeader("X-Forwarded-For");
    if(forwardedFor != null && !forwardedFor..equalsIgnoreCase("unknown")){
        if (forwardedFor.indexOf(",") > 0) {
            return forwardedFor.substring(0, forwardedFor.indexOf(","));
        }
        return forwardedFor;
    }
    String realIp = request.getHeader("X-Real-IP");
    if (realIp != null && !realIp.equalsIgnoreCase("unknown")) {
        return realIp;
    }
    return request.getRemoteAddr();
}
```

#### 6. 关于X-Forwarded-For和X-Real-IP

* `X-Forwarded-For`: `HTTP`协议标准请求头，是用来识别通过`HTTP`代理或负载均衡方式连接到`Web`服务器的客户端最原始的IP地址的`HTTP`请求头字段

```
X-Forwarded-For: client1, proxy1, proxy2, proxy3
```

* `X-Real-IP`: 非`HTTP`标准，多用于`nginx`