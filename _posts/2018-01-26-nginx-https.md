---
layout: post
title: 在nginx中配置https服务
categories: [编程, nginx]
tags: [https]
---

#### 1. nginx安装

`nginx https`依赖`ssl`模块和`openssl-devel`

```
yum install openssl-devel -y

./configure --with-http_ssl_module
```

#### 2. 生成证书和私钥

使用`openssl`生成证书和私钥

```
openssl req -x509 -nodes -days 999 -newkey rsa:2048 -keyout my.key -out my.crt

Generating a 2048 bit RSA private key
.....+++
..............................................+++
writing new private key to 'my.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:GD
Locality Name (eg, city) [Default City]:GZ
Organization Name (eg, company) [Default Company Ltd]:XX
Organizational Unit Name (eg, section) []:XXX
Common Name (eg, your name or your server's hostname) []:www.myapp.com
Email Address []:xx@xx.com
```
- `req -x509`: 生成自签名证书
- `-nodes`: 忽略私钥密码
- `-days 999`: 证书有效期`999`天
- `-newkey rsa:2048`: 生成加密强度为`RSA 2048`的私钥
- `-keyout`: 生成私钥文件名
- `-out`: 生成证书文件名

#### 3. nginx配置
```
server {
    listen       443 ssl;
    server_name  www.myapp.com;

    ssl_certificate      /etc/nginx/my.crt;
    ssl_certificate_key  /etc/nginx/my.key;

    location / {
        root   html;
        index  index.html index.htm;
    }
}
```

#### 4. 代理的http处理

反向代理`http`服务的情况，可通过`filter`把页面响应的`http`协议转换为`https`

```
location ~ \.do$ {
    proxy_pass http://127.0.0.1:8080;
    sub_filter_types html/text;
    sub_filter_once off;
    sub_filter 'http://192.168.1.100:80' 'https://192.168.1.100:443';
    proxy_set_header    REMOTE-HOST $remote_addr;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

> `sub_filter`指令需要安装相应模块`--with-http_ssl_module`