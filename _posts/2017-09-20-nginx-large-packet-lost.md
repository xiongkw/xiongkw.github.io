---
layout: post
title: nginx响应大报文时丢包的问题
categories: [编程, nginx, web]
tags: []
---

> `nginx`反向代理偶尔出现丢包现象

#### 1. 问题
`nginx`反向代理,在返回大报文时出现丢包现象,小报文则不会

#### 2. nginx日志
查看`nginx error`日志
```
open() "/app/nginx/proxy_temp/3/91/0000042913" failed (13: Permission denied) while reading upstream
```

> 说明问题在于没有目录权限

#### 3. 修正
查看`proxy_temp`目录
```
drwx------ 102 nobody nobody 4096 Aug 21 15:06 0
drwx------ 102 nobody nobody 4096 Aug 21 15:07 1
drwx------ 102 nobody nobody 4096 Aug 23 15:38 2
drwxr-xr-x  11 root root  127 Sep 20 12:40 3
drwx------ 102 nobody nobody 4096 Aug 15 22:41 4
```

发现有子目录`3`没有写权限，于是加上权限，丢包问题解决
```
chmod -R 777 3
or
chown -R nobody:nobody 3
```

#### 4. 附

关于`nginx proxy_temp`
> When buffering is enabled, nginx receives a response from the proxied server as soon as possible, saving it into the buffers set by the proxy_buffer_size and proxy_buffers directives. If the whole response does not fit into memory, a part of it can be saved to a temporary file on the disk. Writing to temporary files is controlled by the proxy_max_temp_file_size and proxy_temp_file_write_size directives.
