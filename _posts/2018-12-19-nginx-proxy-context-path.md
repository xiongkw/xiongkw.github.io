---
layout: post
title: nginx反向代理和ContextPath
categories: [编程, nginx]
tags: [proxy, ContextPath]
---

> `ContextPath`是一个`web容器`部署多个应用时为每个应用加入的一个路径，用于区分不同的服务

#### 1. 普通的反向代理
```
location / {
   proxy_pass http://192.168.1.100:8080;
}

```

例如：`http://nginx/a.html`会被代理到`http://192.168.1.100:8080/a.html`

#### 2. 没有ContextPath的访问代理到有ContextPath的后端服务

```
location / {
   proxy_pass http://192.168.1.100:8080/myapp;
}
```

例如：`http://nginx/a.html`会被代理到`http://192.168.1.100:8080/myapp/a.html`

#### 3. 有ContextPath的访问代理到没有ContextPath的后端服务

```
location /myapp {
   proxy_pass http://192.168.1.100:8080/;
}
```

例如：`http://nginx/myapp/a.html`会被代理到`http://192.168.1.100:8080/a.html`


#### 4. 不带斜杠的`proxy_pass`

```
location /myapp {
   proxy_pass http://192.168.1.100:8080;
}
```

例如：`http://nginx/myapp/a.html`会被代理到`http://192.168.1.100:8080/myapp/a.html`

> 原因是不带`/`地址不是一个`URI`，参考[proxy_pass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)