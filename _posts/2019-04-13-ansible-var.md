---
layout: post
title: Ansible复杂变量使用
categories: [编程, linux, ansible]
tags: [ansible]
---


> 使用`ansible`编写自动化安装脚本，发现一些复杂的参数总是重复配置

#### 1. 一个例子



例如`kafka`和`service`都依赖`zookeeper`

hosts
```
[zookeeper]
192.168.1.100
[zookeeper:vars]
zk_port=2181

[kafka]
192.168.1.101
[kafka:vars]
kafka_zk_server=192.168.1.100:2181

[service]
192.168.1.102
[service:vars]
service_zk_server=192.168.1.100:2181
```

> 上面的例子里，`zk_server`配置了相同的值`192.168.1.100:2181`，如果`zookeeper`主机或者端口变化，则这里都要改

#### 2. 一种方法是使用ansible内置变量

inventory/hosts
```
[zookeeper]
192.168.1.100
[zookeeper:vars]
zk_port=2181

[kafka]
192.168.1.101

[service]
192.168.1.102
```

inventory/group_vars/all.yml

```yaml
kafka_zk_server: "\{\{groups['zookeeper'][0]\}\}:\{\{zk_port\}\}"
service_zk_server: "\{\{kafka_zk_server\}\}"
```

#### 3. 改进：使用for循环增加集群支持

inventory/group_vars/all.yml

```yaml
kafka_zk_server: "\{% for host in groups['zookeeper'] %\}\{\{host\}\}:\{\{zk_port\}\}\{% if not loop.last %\},\{% endif %\}\{% endfor %\}"
service_zk_server: "\{\{kafka_zk_server\}\}"
```

#### 参考

* [Ansible中文权威指南](http://www.ansible.com.cn/index.html)

* [Jinja2](http://jinja.pocoo.org/docs/2.10/)