---
layout: post
title: docker修改容器ip网段
categories: [编程, linux, docker]
tags: []
---


> `docker`容器默认的`172`网段和现有网络有冲突，需要修改

#### 1. 修改docker默认网段

```
$ systemctl stop docker
$ vi /etc/docker/daemon.json
{
      "bip" : "192.168.0.1/24"
}
$ systemctl start docker
```

#### 2. 修改docker-compose默认网段

发现通过`docker-compose`起的容器仍然是`172`地址

```
$ systemctl stop docker
$ vi /etc/docker/daemon.json
{
      "bip" : "192.168.0.1/24",
      "default-address-pools" : [
        {
          "base" : "192.168.0.1/16",
          "size" : 24
        }
      ]
}
$ systemctl start docker
```

#### 3. 清理网桥

如果以上修改无效，则可以先清除网桥

```
$ systemctl stop docker
$ ip link set dev docker0 down
$ brctl delbr docker0
$ systemctl start docker
```