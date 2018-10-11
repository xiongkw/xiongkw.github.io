---
layout: post
title: docker registry搭建
categories: [编程, docker, linux]
tags: [docker, 私服]
---


> 由于种种原因，墙内用户总是需要搭建自己的各种私服，例如`nexus`、`npm`、`yum`等，当然`docker`也不例外

#### 1. 安装运行

官方提供了镜像安装的方式，直接拉取即可运行，非常方便

```
$ docker run -d -v /home/docker/.registry:/tmp/registry -p 8500:5000 --restart=always --name registry registry:2

$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                    NAMES
ae5f7208e15f        registry:2          "/entrypoint.sh /e..."   About a minute ago   Up About a minute   0.0.0.0:8500->5000/tcp   elegant_liskov
```

* `-d`:以守护进程运行
* `-v /home/docker/.registry:/var/lib/registry`: 挂载宿主机`/home/docker/.registry`目录到容器`/tmp/registry`，用于保存镜像文件
* `-p 8500:5000`: 映射宿主机`8500`端口到容器`5000`
* `--name registry`: 指定容器名称

#### 2. push

##### 2.1 查看镜像列表

```
$ docker images
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
registry                       2                   2e2f252f3c88        4 weeks ago         33.3 MB
hello-world                    latest              4ab4c602aa5e        4 weeks ago         1.84 kB
```

##### 2.2 给镜像tag

```
$ docker image tag hello-world 192.168.1.100:8500/hello-world

$ docker images
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
registry                       2                   2e2f252f3c88        4 weeks ago         33.3 MB
hello-world                    latest              4ab4c602aa5e        4 weeks ago         1.84 kB
192.168.1.100:8500/hello-world latest              4ab4c602aa5e        4 weeks ago         1.84 kB
```

##### 2.3 push

```
$ docker push 192.168.1.100:8500/hello-world
The push refers to a repository [192.168.1.100:8500/hello-world]
428c97da766c: Pushed
latest: digest: sha256:7d6fb7e5e7a74a4309cc436f6d11c29a96cbf27a4a8cb45a50cb0a326dc32fe8 size: 524
```

##### 2.4 pull

删除原有镜像
```
$ docker rmi 4ab4c602aa5e --force
```

从私服拉取
```
$ docker pull 192.168.1.100:8500/hello-world
```

#### 3. push 报错问题

##### 3.1 Forbidden

```
$ docker push 192.168.1.100:8500/hello-world
The push refers to a repository [192.168.1.100:8500/hello-world]
Get https://192.168.1.100:8500/v1/_ping: Forbidden
```

解决办法：

```
$ vi /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd --insecure-registry 192.168.1.100:8500

$ systemctl daemon-reload
$ systemctl restart docker
```

##### 3.2 manifest blob unknown

```
manifest blob unknown: blob unknown to registry
```

`HTTP`代理导致，去掉代理即可

```
$ vi /usr/lib/systemd/system/docker.service

[Service]

#Environment="HTTP_PROXY=http://xxx" "HTTPS_PROXY=http://xxx"

$ systemctl restart docker
```

#### 4. 参考

* [Docker-Registry](https://github.com/docker/docker-registry)
* [Pip安装方式](https://github.com/docker/docker-registry/blob/master/ADVANCED.md)
* [Docker Distribution](https://github.com/docker/distribution)
* [Docker Registry HTTP API V2](https://github.com/docker/distribution/blob/master/docs/spec/api.md)
* [docker入门]({{site.url}}/2017/02/20/docker-startup)