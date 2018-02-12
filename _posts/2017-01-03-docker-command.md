---
layout: post
title: 常用docker命令
categories: [编程, docker]
tags: [docker]
---

#### `docker version|info`
```
docker version
docker info
```

#### `docker ps`
查看`docker`容器进程
```
docker ps |grep app

kubectl get pod|grep app
```

#### `docker exec`
进入`docker`容器`bash`命令行
```
docker exec -it ${containerid} bash

kubectl exec -it ${containerid} bash
```

#### `docker start|stop|restart|kill`
```
docker start ${containerid}
docker stop ${containerid}
docker restart ${containerid}
docker kill ${containerid}
```

#### `docker cp`
在宿主机和`docker`容器之间`copy`文件
```
docker cp xx ${containerid}:/root/

docker cp ${containerid}:/root/xx ./
```

#### 找出`docker`容器在宿主机上对应的虚拟网卡
```
route -n |grep ${containerip}|awk '{print $NF}'
```


