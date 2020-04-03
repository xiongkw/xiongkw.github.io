---
layout: post
title: docker镜像搭建nfs-server
categories: [编程, linux, docker]
tags: [nfs-server]
---


> 搭建一个`nfs`服务通常需要安装`nfs-utils、rpcbind`等，本文直接使用镜像来创建

#### 1. 拉取镜像

使用`gists/nfs-server`镜像

```
$ docker pull gists/nfs-server:2.4.1
$ docker images
```

#### 2. 启动容器

使用`docker-compose`启动容器

```yaml
version: '2'

services:
  nfs-server:
    container_name: nfs-server
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "128m"
    image: gists/nfs-server:2.4.1
    cap_add:
      - SYS_ADMIN
      - SETPCAP
    environment:
      TZ: Asia/Shanghai
      NFS_OPTION: "fsid=0,rw,sync,insecure,all_squash,anonuid=65534,anongid=65534,no_subtree_check,nohide"
    volumes:
      - /nfs-share:/nfs-share #需要先创建/nfs-share目录并且设置用户的读写权限
    ports:
      - 2049:2049
```

启动

```
$ docker-compose up -d
```

#### 3. 客户端挂载

```
$ mkdir /data/share
$ sudo mount -v -t nfs -o vers=4,port=2049 192.168.1.100:/ /data/share
$ df -Th
Filesystem              Type      Size  Used Avail Use% Mounted on
192.168.1.100:/         nfs4      502G   68G  434G  14% /data/share
```

#### 4. 问题

`showmount`命令执行错误

```
$ showmount -e 192.168.1.100
clnt_create: RPC: Program not registered
```

进入容器，启动`rpc.mountd`就能查看了

```
$ docker exec -it nfs-server /bin/sh
/ # rpc.mountd
/ # showmount -e
Export list for f3d25bc825d1:
/nfs-share *
```

#### 5. 参考

* [gists/nfs-server](https://hub.docker.com/r/gists/nfs-server)
* [Runtime privilege and Linux capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)