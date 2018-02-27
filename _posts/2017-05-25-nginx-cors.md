---
layout: post
title: nginx反向代理开启cors
categories: [编程, nginx]
tags: [cors, 跨域]
---


> 在`nginx`反向代理中配置`cors`跨域资源访问

#### 1. 配置

```
server {
        listen 80;
        root /usr/local/nginx/html;
        server_name a.test.com;

        location / {

                if ($request_method = 'OPTIONS') {
                    add_header Access-Control-Allow-Origin *;
                    add_header Access-Control-Allow-Headers x-requested-with,content-type,requesttype;
                    add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,OPTIONS;
                    return 204;
                }
                add_header Access-Control-Allow-Origin *;
                proxy_pass HTTP://a-test-com;
        }
}
```

#### 2. 参考

* [HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
* [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)
* [jsonp和cors]({{site.url}}/2016/12/15/jsonp-cors/)