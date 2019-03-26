---
layout: post
title: Harbor安装
categories: [编程, linux, docker]
tags: [harbor]
---


> `Harbor`是一个企业级的镜像仓库，类似`nexus`之于`maven`

#### 1. 下载安装包

下载`offline`安装包，[goharbor/harbor](https://github.com/goharbor/harbor/releases)

#### 2. 安装docker依赖

`Harbor`依赖`docker-compose`和`docker`

```
$ yum install docker docker-compose
```

#### 3. 解压安装

```
$ tar zxvf harbor-offline-installer-v1.7.4.tgz
$ cd harbor

```

#### 4. 修改配置

编辑harbor.cfg

```
hostname = 192.168.1.100:8000
```

编辑docker-compose.yml
```
#修改registry镜像
registry:
  image: registry:2.5.0

#修改nginx端口
proxy
  ports:
    - 8000:80
```

> 这里的`8000`端口对应`hostname`中的端口

#### 5. 修改数据目录

`Harbor`默认数据目录为`/data`，我们可以选择自定义目录

编辑harbor.cfg
```
ssl_cert = //data/harbor/cert/server.crt
ssl_cert_key = //data/harbor/cert/server.key
```

编辑docker-compose.yml
```
#修改镜像挂载的宿主机目录

*
  volumes:
    - //data/harbor/*
```

#### 6. 去掉push时用户认证

修改common/templates/registry/config.yml，删除如下几行
```
auth:
  token:
    issuer: registry-token-issuer
    realm: $ui_url/service/token
    rootcertbundle: /etc/registry/root.crt
    service: token-service
```

#### 7. 安装并启动

```
# 关闭selinux
$ ./install.sh
```

#### 8. 查看启动的容器

```
      Name                     Command               State               Ports             
------------------------------------------------------------------------------------------
harbor-db           docker-entrypoint.sh mysqld      Up      3306/tcp                      
harbor-jobservice   /harbor/harbor_jobservice        Up                                    
harbor-log          /bin/sh -c crond && rm -f  ...   Up      0.0.0.0:1514->514/tcp         
harbor-ui           /harbor/harbor_ui                Up                                    
nginx               nginx -g daemon off;             Up      443/tcp, 0.0.0.0:8021->80/tcp 
registry            /entrypoint.sh serve /etc/ ...   Up      5000/tcp
```

* `harbor-db`：`mysql`数据库
* `harbor-jobservice`：`job`管理模块，主要用于仓库之间同步镜像
* `harbor-log`：日志服务
* `harbor-ui`：`web`管理页面，主要是前端的页面和后端`rest`接口;
* `nginx`：负责流量转发和安全验证，对外提供的流量都是从`nginx`中转
* `registry`：`docker`原生的镜像仓库，负责保存镜像

#### 9. harbor管理

```
# 停止
$ docker-compose down
# 启动
$ docker-compose up -d

```

#### 10. 镜像上传

修改`docker client`镜像仓库地址
```
$ vim /usr/lib/systemd/system/docker.service
# 增加仓库 --insecure-registry 192.168.1.100:8000
$ ExecStart=/usr/bin/dockerd --insecure-registry 192.168.1.100:8000
$ systemctl daemon-reload
$ systemctl restart docker
```

使用`harbor`的`nginx`镜像构建一个新的镜像
```
$ docker images
REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
vmware/harbor-log                   0.5.0               8a0833f24c8f        2 years ago         191 MB
vmware/harbor-jobservice            0.5.0               f8d65542009e        2 years ago         169 MB
vmware/harbor-ui                    0.5.0               587e09accc1b        2 years ago         233 MB
vmware/harbor-db                    0.5.0               e4339a680042        2 years ago         327 MB
nginx                               1.11.5              05a60462f8ba        2 years ago         181 MB
registry                            2.5.0               c6c14b3960bd        2 years ago         33.3 MB
```

编写Dockerfile
```
$ echo "Hello harbor" > a.txt
$ vi Dockerfile
FROM nginx:1.11.5

ADD a.txt /root
```

构建镜像和打tag
```
$ docker build t nginxx:0.1 .
$ docker tag nginxx:0.1 192.168.1.100:8000/library/nginxx:0.1
$ docker images
REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
192.168.1.100:8000/library/nginxx   0.1                 8a3040e8194c        4 minutes ago       181 MB
nginxx                              0.1                 8a3040e8194c        4 minutes ago       181 MB
```

上传到harbor
```
docker push 192.168.1.100:8000/library/nginxx:0.1
```

在harbor管理页面查看，省略...

#### 11. 参考

* [Harbor](https://github.com/goharbor/harbor)