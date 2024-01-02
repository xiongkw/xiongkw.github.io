---
layout: post
title: Debian10安装docker-ce
categories: [linux]
tags: [docker]
---

> 

#### 1. 添加软件源

```
$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/debian $(lsb_release -cs) stable"
```

#### 2. 安装docker-ce

```
$ sudo apt-get install docker-ce
$ sudo systemctl start docker
```

#### 3. 添加用户权限

```
$ sudo groupadd docker
$ newgrp docker
```

#### 4. 设置镜像源

/etc/docker/daemon.json
```
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

#### 5. 修改数据目录

/etc/docker/daemon.json
```
{
	"data-dir": "/data/docker/data"
}
```

#### 6. 下载镜像

```
$ docker search alpine
$ docker pull alpine
```

#### 7. 查询指定镜像的Tag列表

```
#!/bin/bash

repo_url=https://registry.hub.docker.com/v2/repositories/library
image_name=$1

curl -L -s ${repo_url}/${image_name}/tags?page_size=1024 | jq '.results[]["name"]' | sed 's/\"//g' | sort -u
```