---
layout: post
title: nginx中通过文本过滤和cookie重写实现ip端口到内部域名系统的代理
categories: [编程, nginx, web]
tags: [ip, cookie, proxy]
---

> 微服务架构下，采用`lvs+nginx`做负载均衡，对外提供域名访问，子应用通过二级域名划分，同时各子应用之间做了[基于cookie共享实现单点登录]({{ site.url }}/2016/04/01/single-sign-on-cookie/)   
> 由于一些网络原因，有些营业厅需要通过`dcn、vpn`等方式访问，所以需要提供`ip+port`的访问方式

#### 1. 问题
现有环境：
```
应用1: app1.xx.com
应用2: app2.xx.com
cookie domain: xx.com
lvs vip: 192.168.1.230
```

需要配置为`ip`端口访问：
```
nginx ip: 192.168.1.220
应用1: 192.168.1.220:801
应用2: 192.168.1.220:802
```

#### 2. 配置nginx端口转发
```
server {
        listen       801;
        server_name  localhost;
        location / {
            proxy_set_header HOST app1.xx.com;
            proxy_pass HTTP://192.168.1.230;
        }

    }

    server {
        listen       802;
        server_name  localhost;
        location / {
            proxy_set_header HOST app2.xx.com;
            proxy_pass HTTP://192.168.1.230;
        }

    }
```

#### 3. 配置cookie重写
```
        location / {
            proxy_set_header Cookie $http_cookie;
            proxy_cookie_domain xx.com 192.168.1.220;
        }
```

#### 4. 配置请求响应文本中的域名替换为ip

由于菜单中配置的是域名，所以需要替换为ip端口
```
        location = /sys/menus {
            sub_filter_types application/json;
            sub_filter_once off;
            sub_filter 'app1.xx.com' '192.168.1.220:801';
            sub_filter 'app2.xx.com' '192.168.1.220:802';
            proxy_set_header HOST app1.xx.com;
            proxy_set_header Cookie $http_cookie;
            proxy_cookie_domain xx.com 192.168.1.220;
            proxy_pass HTTP://192.168.1.230;
        }
```

#### 5. 完整配置
```
    server {
        listen       801;
        server_name  localhost;
        
        location = /sys/menus {
            sub_filter_types application/json;
            sub_filter_once off;
            sub_filter 'app1.xx.com' '192.168.1.220:801';
            sub_filter 'app2.xx.com' '192.168.1.220:802';
            proxy_set_header HOST app1.xx.com;
            proxy_set_header Cookie $http_cookie;
            proxy_cookie_domain xx.com 192.168.1.220;
            proxy_pass HTTP://192.168.1.230;
        }

        location / {
            proxy_set_header HOST app1.xx.com;
            proxy_set_header Cookie $http_cookie;
            proxy_cookie_domain xx.com 192.168.1.220;
            proxy_pass HTTP://192.168.1.230;
        }

    }

    server {
        listen       802;
        server_name  localhost;
        
        location = /sys/menus {
            sub_filter_types application/json;
            sub_filter_once off;
            sub_filter 'app1.xx.com' '192.168.1.220:801';
            sub_filter 'app2.xx.com' '192.168.1.220:802';
            proxy_set_header HOST app1.xx.com;
            proxy_set_header Cookie $http_cookie;
            proxy_cookie_domain xx.com 192.168.1.220;
            proxy_pass HTTP://192.168.1.230;
        }
        
        location / {
            proxy_set_header HOST app2.xx.com;
            proxy_set_header Cookie $http_cookie;
            proxy_cookie_domain xx.com 192.168.1.220;
            proxy_pass HTTP://192.168.1.230;
        }

    }
```

#### 6. 注意

> 安装`nginx`时需要指定`sub_filter`模块`--with-http_sub_module`   
> `nginx-1.9.4`以前的版本不支持多个`sub_filter`指令