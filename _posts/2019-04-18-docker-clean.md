---
layout: post
title: docker清理命令
categories: [编程, linux, docker]
tags: []
---


> `docker`主机上有大量镜像、停止的容器等需要清理

#### 清理命令

```
# kill 所有容器
$ docker kill $(docker ps -a -q)

# 删除所有停止容器
$ docker rm $(docker ps -a -q)

# 删除所有镜像
$ docker rmi $(docker images -q) -f

```