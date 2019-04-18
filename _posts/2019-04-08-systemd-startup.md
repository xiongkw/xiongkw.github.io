---
layout: post
title: Systemd系统服务管理
categories: [编程, linux]
tags: [systemd]
---


> 早几年前，我常用`init.d`的方式编写`linux`系统服务，现在有了`systemd`

#### 1. 常用命令

```
$ systemctl start httpd  // service httpd start
$ systemctl stop httpd  // service httpd stop
$ systemctl status httpd  // service httpd status
```

#### 2. 开机启动

```
$ systemctl enable httpd // chkconfig httpd on
$ systemctl disable httpd // chkconfig httpd off
```

#### 3. 编写一个systemd服务

一个`EalsticSerach`的例子

/usr/lib/systemd/system/elasticsearch.service

```
[Unit]
Description=Elasticsearch Server
After=network.target
Wants=network.target

[Service]
Type=forking
Environment=JAVA_HOME=/app/jdk1.8.0_201
ExecStart=/app/elasticsearch-6.2.4/bin/elasticsearch -d -p /app/elasticsearch-6.2.4/PID
Restart=on-failure
LimitNOFILE=655350
LimitNPROC=20480
User=elasticsearch
TimeoutSec=300

[Install]
WantedBy=multi-user.target

```

#### 参考

* [Systemd 入门教程：实战篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html)