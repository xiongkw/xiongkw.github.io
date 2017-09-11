---
layout: post
title: ab压力测试中碰到的问题和解决方法
categories: [编程, 压力测试]
tags: [ab]
---

##### 1. socket: Too many open files(24)

原因： 打开文件(socket)太多，一般是由于并发数大于用户可打开文件数引起

解决方法：调整客户机用户可打开文件数
```
ulimit -n 65535
```
或者修改
```
vi /etc/security/limits.conf

*       soft    nofile  65535  
*       hard    nofile  65535  
```

##### 2. apr_socket_recv: Connection reset by peer (104)

原因：连接被重置了，一般由于服务器开启了反洪水攻击

解决方法：禁用服务器的反洪水攻击或者启用`ab -r`参数
```
sudo sysctl net.ipv4.tcp_syncookies=0
```
或者
```
vi /etc/sysctl.conf

net.ipv4.tcp_syncookies=0

sysctl -p
```

##### 3. apr_socket_recv: Connection timed out (110)

原因：连接超时，一般是由于压力过大导致服务器处理不过来引起

解决方法：减小并发压力或者启用`ab -r`参数


##### 4. 如何知道请求是否成功()查看响应信息)

增加参数`ab -v 4`打印响应信息
```
ab -v 4
```

##### 5. 如何启用keepalive
```
ab -k
```

##### 6. 如何发起HTTP 1.1请求
`ab 2.3`版本只支持`HTTP 1.0`请求，如果要发起`HTTP 1.1`请求，可以使用其它压测工具(例如`jmeter`)