---
layout: post
title: Apache Jmeter的远程模式
categories: [java, linux, 压力测试]
tags: []
---

> 使用Jmeter做压测，压测客户机为linux服务器，为了能在本机通过GUI操控，所以尝试Remote模式

#### 1. 远程服务端

安装Jmeter，修改bin/jmeter.properties
```
server_port=10099

server.rmi.ssl.disable=true
```

启动server，有多块网卡时，指定rmi绑定的hostname

```
$ bin/jmeter-server -Djava.rmi.server.hostname=192.168.1.100
```

#### 2. 控制端

安装Jmeter，修改bin/jmeter.properties
```
remote_hosts=192.168.1.100:10099

server.rmi.ssl.disable=true
```

启动控制端，以windows为例

```
$ bin/jmeter.bat
```

#### 3. 测试

编写测试计划，以Remote方式运行

Run-Remote Start-192.168.1.100:10099

测试计划可在服务端正常执行，并能通过本地GUI查看测试结果

#### 4. 注意事项

* 控制端通过RMI调用发送和运行测试计划到服务端
* 服务端通过RMI调用发送结果报告给控制端

所以要求网络是双向互通，注意关闭防火墙

#### 5. 网络只能单向访问时的处理方法

一般都是这种情况：本地电脑通过代理等方式访问服务器，服务器不能直接访问本地电脑

解决思路是通过SSH隧道做反向端口转发

##### 5.1 设置控制端RMI地址

在jmeter.bat中增加参数-Djava.rmi.server.hostname=127.0.0.1，表示服务端对控制端RMI调用的IP地址

##### 5.2 设置本地RMI端口

```
client.rmi.localport=10199
```

##### 5.3 反向端口转发

通过SSH隧道做反向端口转发，以SecureCRT为例：控制机直接SSH到服务端

Session Options-Connection-Port Forwarding-Remote/X11

```
Name   Remote Address    Local Host
10199     10199      127.0.0.1:10199
10200     10200      127.0.0.1:10200
10201     10201      127.0.0.1:10201
```

注意：这里要开三个端口，原因如下：

```
JMeter/RMI requires a connection from the client to the server. This will use the port you chose, default 1099.
JMeter/RMI also requires a reverse connection in order to return sample results from the server to the client.
These will use high-numbered ports.
These ports can be controlled by jmeter property called client.rmi.localport in jmeter.properties.
If this is non-zero, it will be used as the base for local port numbers for the client engine. At the moment JMeter will open up to three ports beginning with the port defined in client.rmi.localport. If there are any firewalls or other network filters between JMeter client and server, you will need to make sure that they are set up to allow the connections through. If necessary, use monitoring software to show what traffic is being generated.


```

#### 参考

* [Remote Testing](https://jmeter.apache.org/usermanual/remote-test.html)