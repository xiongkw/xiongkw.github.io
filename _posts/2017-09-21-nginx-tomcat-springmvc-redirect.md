---
layout: post
title: nginx代理tomcat时spring-mvc中的redirect问题
categories: [编程, nginx, spring]
tags: [spring-mvc, redirect]
---

> spring-mvc中退出登录后要重定向到用户访问的初始请求页面

#### 1. 实现原理

1. 保存初始请求的`queryString`到用户`session`中

```java
@RequestMapping(value = "/hello", method = RequestMethod.GET)
public String hello(@RequestParam String name, HttpSession session, HttpServletRequest request) {
    //
    session.setAttribute(IConstants.SESSION_KEY_ORIGIN_REQUEST_QUERY_STRING, request.getQueryString());
    //  
}
```

2. 退出登录前取出`queryString`，退出后重定向到初始请求`controller`

```java
@RequestMapping(value = "/logout", method = RequestMethod.GET)
public String logout(HttpSession session) {
    String queryString = (String) session.getAttribute(IConstants.SESSION_KEY_ORIGIN_REQUEST_QUERY_STRING);
    session.invalidate();
    return "redirect:/hello?" + queryString;
}
```

#### 2. 异常现象

在本地`Tomcat`调试能正常重定向，但在`nginx`反向代理后无法重定向，查看url地址发现缺少端口号

#### 3. 解决方法

配置`nginx`反向代理：

```
location / {
    proxy_set_header Host $host:$server_port;
    
}
```

> Catalina's HttpServletResponse.sendRedirect implementation uses the getServerPort method to build an absolute redirect url (Location Header-Value). getServerPort returns the part after the : from the request's HostHeader-Value which has to be 8085 in your case.

#### 4. 参考
[Spring MVC “redirect:/” prefix redirects with port number included](https://stackoverflow.com/questions/20587132/spring-mvc-redirect-prefix-redirects-with-port-number-included)

