---
layout: post
title: nginx性能优化实践
categories: [编程, nginx]
tags: [性能优化]
---

#### 1. 全局核心配置

##### 1.1 worker_processes

`worker`进程数，一般设置为`CPU`核数可以设置为`auto`

##### 1.2 worker_rlimit_nofile

`worker`进程的最大打开文件数限制，一般可设置为`65536`

#### 2 Events模块配置

##### 2.1 use参数

一般`Linux`生产环境设置为`epoll`

##### 2.2 worker_connections

一个`worker`进程同时打开的最大连接数，默认是1024，一般设置为65536

##### 2.3 multi_accept

尽可能多接受连接请求，设置为`on`

#### 3 Http模块配置

##### 3.1 **sendfile**参数

设置为`on`

可以在磁盘和`TCP socket`之间互相拷贝数据，`sendfile()`要比组合`read()`和`write()`以及打开关闭丢弃缓冲更加有效

##### 3.2 tcp_nopush参数

设置为`on`

告诉`nginx`在一个数据包里发送所有头文件，而不一个接一个的发送

##### 3.3 tcp_nodelay参数

设置为`on`

告诉`nginx`不要缓存数据，而是一段一段的发送--当需要及时发送数据时，就应该给应用设置这个属性，这样发送一小块数据信息时就不能立即得到返回值

##### 3.4 access_log参数

设为`off`

写访问日志,如果生产系统有其它的审计功能，可以关闭该参数，减少磁盘IO读写

##### 3.5 error_log参数

建议设置为`warn`

##### 3.6 gzip参数

设置为`on`

采用`gzip`压缩的形式发送数据

##### 3.7 gzip_comp_level参数

设置数据的压缩等级，可以是1-9之间的任意数值，9是最慢但是压缩比最大的

一般设置为4，这是一个比较折中的设置

##### 3.8 gzip_types参数
```
text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
```

##### 3.9 gzip_min_length参数

设为`1024`

##### 3.10 gzip_buffers参数

设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流.
```
4 8k;
```

##### 4.11 client_max_body_size参数
```
100M
```

##### 4.12 client_body_buffer_size参数
```
128K
```

##### 3.13 proxy_connect_timeout参数

设置为`90`

`nginx`跟后端服务器连接超时时间(代理连接超时)。

##### 3.14 proxy_send_timeout参数

设置为`90`

后端服务器数据回传时间(代理发送超时)。

##### 3.15 proxy_read_timeout参数

设置为`90`

连接成功后，后端服务器响应时间(代理接收超时)

##### 3.16 proxy_buffer_size参数
```
4k;
```

##### 4.17 proxy_buffers
```
4  32k;
```

##### 3.18 proxy_busy_buffers_size

高负荷下缓冲大小

一般设置为（`proxy_buffers*2`）

#### 4. 长连接设置

##### 4.1 客户端长连接

###### 4.1.2 keepalive_timeout

默认75s，给客户端分配`keep-alive`链接超时时间，设置低些可以加快连接回收速度

###### 4.1.2 keepalive_requests

设置一个客户端长连接可以处理的最大请求数，超出后连接将会被关闭，默认`100`

##### 4.2 反向代理长连接

###### 4.2.1 proxy_http_version

设置代理`http`版本，设置为`1.1`，默认为`1.0`

```
proxy_http_version 1.1;
```

###### 4.2.2 Connection

设置`Connection`头信息为空，默认为`close`

```
proxy_set_header Connection "";
```

###### 4.2.3 keepalive

激活对上游服务器的连接进行缓存。`connections`参数设置每个`worker`进程与后端服务器保持连接的最大数量。这些保持的连接会被放入缓存。如果连接数大于这个值时，最久未使用的连接会被关闭。需要注意的是，`keepalive`指令不会限制`Nginx`进程与上游服务器的连接总数。 新的连接总会按需被创建。`connections`参数应该稍微设低一点，以便上游服务器也能处理额外新进来的连接

```
keepalive 10000;
```

#### 5. 查看Nginx状态配置

##### 5.1 设置允许查看Nginx状态

```
location /NginxStatus {
stub_status on;
access_log on;
auth_basic "NginxStatus";
auth_basic_user_file conf/htpasswd;
#htpasswd文件的内容可以用apache提供的htpasswd工具来产生

}
```
