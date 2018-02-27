---
layout: post
title: 常用docker命令
categories: [编程, docker]
tags: [docker]
---

#### 1. 查看版本信息

```
docker version
docker info
```

#### 2. 查看进程

```
docker ps |grep app

kubectl get pod|grep app
```

#### 3. 进入容器命令行

```
docker exec -it ${containerid} bash

kubectl exec -it ${containerid} bash
```

#### 4. 容器相关操作
```
docker start ${containerid}
docker stop ${containerid}
docker restart ${containerid}
docker kill ${containerid}
```

#### 5. 文件复制

在宿主机和`docker`容器之间`copy`文件
```
docker cp xx ${containerid}:/root/

docker cp ${containerid}:/root/xx ./
```

#### 6. 网卡

找出`docker`容器在宿主机上对应的虚拟网卡
```
route -n |grep ${containerip}|awk '{print $NF}'
```


