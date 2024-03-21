---
layout: post
title: docker推送镜像到harbor外网地址的问题
categories: [编程, linux]
tags: []
---

>  docker push镜像时卡住了

#### 1. 现象

docker镜像push到harbor时卡住了

```
$ docker push 180.119.140.230:8200/libarary/web:1.1.0
The push refers to repository [180.119.140.230:8200/libarary/web:1.1.0]
dd09b7de8e9e: Preparing 
5edc59dff9d6: Preparing 
8c40ec2bdd9d: Preparing 
```

#### 2. 抓包分析

使用tcpdump抓包

```
$ tcpdump -i eth0 port 8200 -w a.cap
```

使用wireshark分析关键包

```

17	2.442875	192.168.1.100	180.119.140.230	HTTP	325	GET /v2/ HTTP/1.1 

20	2.472529	180.119.140.230	192.168.1.100	HTTP	794	HTTP/1.1 401 Unauthorized  (application/json)
Yo'HTTP/1.1 401 Unauthorized
Server: unknown
Date: Thu, 23 Jun 2022 04:12:43 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 87
Connection: close
Docker-Distribution-Api-Version: registry/2.0
Set-Cookie: sid=1735c3ed0bdb2a3d6f9b4dcba7eed257; Path=/; HttpOnly
Set-Cookie: _xsrf=VkJyYm5xYkFDM3lhU1R1ZElaS3M1UkgwblRubDBmaTI=|1655957569412034198|ca6e6c9b16e764449a4dfe0fe7ef4842d7d2551257f5fabf1e8182bb7b32943a; Expires=Thu, 23 Jun 2022 05:12:49 UTC; Max-Age=3600; Path=/
Www-Authenticate: Bearer realm="http://172.16.200.100:8200/service/token",service="harbor-registry"
unique-id: 1f5e7df45053042ae89b686edbe776d9
{"errors":[{"code":"UNAUTHORIZED","message":"authentication required","detail":null}]}

26	2.623993	192.168.1.100	172.16.200.100	TCP	74	[TCP Retransmission] 47028→8200 [SYN] Seq=0 Win=42340 Len=0 MSS=1460 SACK_PERM=1 TSval=3761639668 TSecr=0 WS=8192
```

可以看到docker客户端先发了一个`GET /v2/`请求(17)，harbor响应了401并返回了一个地址`http://172.16.200.100:8200/service/token`(20)，docker再发请求到`172.16.200.100:8200`(26)，而docker所在主机到172.16.200.100的网络不通，所以发生了`TCP Retransmission`

#### 4. 问题原因

docker请求harbor时，如果认证失败，会去请求一个serviceToken，而harbor返回了一个内网地址的api地址

#### 3. 解决办法

##### 3.1 修改harbor配置，返回外网api地址

略

##### 3.2 在docker客户端主机做端口转发

把所有出口172.16.200.100:8200的ip包转发到180.119.140.230:8200

```
$ iptables -t nat -I OUTPUT -d 172.16.200.100 -p tcp --dport 8200 -j DNAT --to 180.119.140.230:8200
```

