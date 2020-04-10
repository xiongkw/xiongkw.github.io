---
layout: post
title: 常用docker命令
categories: [编程, docker]
tags: [docker]
---

#### 1. 查看版本信息

```
$ docker version
$ docker info
```

#### 2. 查看容器进程

```
$ docker ps |grep app

$ kubectl get pod|grep app
```

#### 3. 打开容器交互命令行

```
$ docker exec -it ${containerid} bash

$ kubectl exec -it ${containerid} bash
```

#### 4. 容器起停相关操作
```
$ docker start ${containerid}
$ docker stop ${containerid}
$ docker restart ${containerid}
$ docker kill ${containerid}
```

#### 5. 容器和宿主机之间的文件复制

在宿主机和`docker`容器之间`copy`文件
```
$ docker cp xx ${containerid}:/root/

$ docker cp ${containerid}:/root/xx ./
```

#### 6. 查看容器对应网卡

找出`docker`容器在宿主机上对应的虚拟网卡
```
$ route -n |grep ${containerip}|awk '{print $NF}'
```

#### 7. 查看容器进程对应的物理进程ID

1. 在`cgroup`文件中查找
```
$ docker ps
$ cat /sys/fs/cgroup/memory/docker/${containerId}/cgroups.procs
```

2. 使用`docker top`命令
```
$ docker top ${containerId}
```

3. 使用`docker inspect`命令
{% raw %}
```
$ docker inspect -f '{{.State.Pid}}' ${containerId}
```
{% endraw %}