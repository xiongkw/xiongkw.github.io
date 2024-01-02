---
layout: post
title: nginx开启Basic Auth
categories: [linux]
tags: [nginx]
---

> 

#### 1. 安装ngx_http_auth_basic_module模块
nginx默认会安装ngx_http_auth_basic_module模块，不需要再指定


#### 2. 生成认证文件

使用htpasswd命令生成认证文件
```
# 安装httpd-tools
$ yum install httpd-tools -y
# 生成认证文件
$ htpasswd -c -d $nginx_home/conf/htpasswd admin
# 输入密码
```

#### 3. 配置nginx

```
http {
	server {
		listen       8080;
		server_name  localhost;

		auth_basic "Please enter your username and password";
		auth_basic_user_file /app/nginx/conf/htpasswd;
		...
	}
}

```