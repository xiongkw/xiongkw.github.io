---
layout: post
title: python私有源搭建
categories: [编程, linux]
tags: [yum]
---

> 有一堆机器，无法连外网，所以也需要搭建本地的`pip`私有源，以`CentOS7`为例

#### 1. 安装pypiserver

##### 1.1 安装pip

通过`pip`安装`pypiserver`，所以需要先安装`pip`

```
$ yum install python2-pip -y
```

##### 1.2 安装pypiserver

```
$ pip install pypiserver
```

#### 2. 下载依赖包

```
$ mkdir pypi
$ pip download -d pypi pypiserver
```

#### 3. 启动pypi-server

```
$ nohup pypi-server -p 8000 pypi >/dev/null 2>&1 &
```

#### 4. 私有源的使用

##### 4.1 命令行指定

```
$ pip install --trusted-host 192.168.1.101 -i http://192.168.1.101:8000 pypiserver
```

##### 4.2 配置文件指定

```
$ vi ~/.pip/pip.conf
[global]
trusted-host = 192.168.1.101
index-url = http://192.168.1.101:8000
$ pip install pypiserver
```

#### 5. 附

##### 5.1 pip源镜像设置

`pip`官方的源下载速度太慢，可以换成阿里云的镜像

```
$ vi ~/.pip/pip.conf
[global]
trusted-host = mirrors.aliyun.com
index-url = https://mirrors.aliyun.com/pypi/simple
```

##### 5.2 批量依赖包下载

使用文本管理依赖包

```
$ vi requirements.txt
nova
keystone
```

下载依赖包

```
$ pip download -r requirements.txt -d pypi
```