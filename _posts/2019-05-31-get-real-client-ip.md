---
layout: post
title: 在复杂网络环境下如何获取客户端真实IP
categories: [编程, linux]
tags: [realip，proxy_protocol]
---


> 使用`nginx`的`stream`方式反向代理`Tomcat`，服务端获取到的客户端IP为`nginx`主机的IP

#### java获取客户端真实ip逻辑

这是一段常见的获取客户端真实ip的`java`代码

```
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

> 参考[Java中获取HTTP请求的Client端真实IP]({{site.url}}/2019/01/12/http-real-ip/)

#### 1. 网络直连

```
+--------+           +--------+
| Broser |  -------> | Tomcat |
+--------+           +--------+
```

客户端(浏览器)和服务端能够直连(没有经过任何代理转发)时，以上代码能够正确获取到客户端IP(通过`request.getRemoteAddr()`)


#### 2. 经过了HTTP代理

现在情况变得稍微复杂一些，中间经过了`HTTP`代理(`nginx`)

```
+--------+           +--------+   HTTP    +--------+
| Broser |  -------> | nginx  | --------> | Tomcat |
+--------+           +--------+           +--------+
```

可以通过设置转发的`HTTP`请求头来实现

```
location / {
    #...
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

以上`nginx`配置会修改原始`HTTP`请求的两个`header`，并转发到后端服务，所以我们的代码可以通过`X-Forwarded-For`或者`X-Real-IP`获取到正确的客户端IP

#### 3. 经过了TCP代理

`nginx TCP`代理原理

```
+--------+           +--------+   TCP     +--------+
| Broser |  -------> | nginx  | --------> | Tomcat |
+--------+           +--------+           +--------+
```

现在，代码获取到的客户端IP为`nginx`代理机的IP，原因是`nginx TCP`代理时修改了IP包的`src`为`nginx`主机的IP，最终程序通过`request.getRemoteAddr()`获取到的正是`nginx`主机的地址

##### 3.1 IP transparent

`nginx`提供一种方法[proxy_bind](http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html#proxy_bind)，能够不修改`src ip`，但是需要修改被代理服务主机的默认网关为`nginx`主机

问题是被代理服务和`nginx`在同一主机上，更改默认网关为自己会有问题

> 参考[IP Transparency and Direct Server Return with NGINX and NGINX Plus as Transparent Proxy](https://www.nginx.com/blog/ip-transparency-direct-server-return-nginx-plus-transparent-proxy/)

##### 3.2 proxy protocl

查看`nginx`文档，发现可以通过`proxy_protocol`来传递客户端真实IP，所以改进如下

```
+--------+           +---------+   TCP     +--------+   TCP     +--------+
| Broser |  -------> | haproxy | --------> | nginx  | --------> | Tomcat |
+--------+           +---------+           +--------+           +--------+
```

其中haproxy配置:

```
server          server1 192.168.1.100:8080 send-proxy
```

> `send-proxy`即启用`proxy protocol`转发请求，其实是客户端IP等信息加到了TCP协议头中

nginx配置:

```
server {
        listen       8080 proxy_protocol;      
}
```

> `proxy_protocol`即指定接收到的是`proxy protocol`，可以从TCP协议头中解析出客户端IP

到目前为止，可以在nginx中获取到真实客户端IP，可通过`$proxy_protocol_addr`变量打印在log中

```
log_format basic '$proxy_protocol_addr $remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time';

access_log /home/docker/dwf/nginx152/logs/access.log basic;
```

但是由于`nginx`到`tomcat`走的是`TCP`转发，而`Tomcat`又不支持`proxy protocol`，所以在`Tomcat`中依然获取不到正确的地址

#### 4. 解决方法

其实最优的解法是改`nginx`反向代理为`HTTP`方式，但是由于种种原因，这个方法无法实行

其次是`IP transparent`，但是不适合`nginx`与被代理服务在相同主机的情况

#### 4.1 客户端上报

好在客户端也能够编程，所以可以在客户端获取到再上报到服务端，例如使用请求头`X-Client-IP`

缺点是只能获取到内网地址，而内网地址对于服务端来说没有任何意义，所以只适用于客户端和服务端在同一局域网内的情况

#### 4.2 结合X-Forwarded-For

综合客户端与服务端获取两种方式，通过以下优先级获取客户端IP

1. X-Forwarded-For
2. X-Real-IP
3. X-Client-IP
4. remoteAddr

> 如果`X-Forwarded-For`有值，说明经过了其它代理，而`X-Forwarded-For`中记录的是客户端外网地址，是有效地址   
> 如果`X-Forwarded-For`无值，说明客户端是可以直连到服务端，其内网地址`X-Client-IP`也是有效地址

#### 参考

* [Module ngx_stream_realip_module](http://nginx.org/en/docs/stream/ngx_stream_realip_module.html#set_real_ip_from)
* [Accepting the PROXY Protocol](https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/#)
* [proxy_bind](http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html#proxy_bind)
* [IP Transparency and Direct Server Return with NGINX and NGINX Plus as Transparent Proxy](https://www.nginx.com/blog/ip-transparency-direct-server-return-nginx-plus-transparent-proxy/)
