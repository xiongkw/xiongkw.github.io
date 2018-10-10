---
layout: post
title: docker入门
categories: [编程, docker]
tags: [docker]
---

#### 1. 简介

> `Docker` 是一个开源的应用容器引擎，基于 `Go `语言 并遵从`Apache2.0`协议开源。 `Docker` 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 `Linux` 机器上，也可以实现虚拟化。 容器是完全使用沙箱机制，相互之间不会有任何接口（类似 `iPhone` 的 `app`）,更重要的是容器性能开销极低。

应用场景

* `Web` 应用的自动化打包和发布
* 自动化测试和持续集成、发布
* 在服务型环境中部署和调整数据库或其他的后台应用
* 从头编译或者扩展现有的`OpenShift`或`Cloud Foundry`平台来搭建自己的`PaaS`环境

![]({{site.url}}/public/images/2017-02-20-docker-startup.png)

`docker`容器以操作系统进程方式运行，所以相比虚拟机更为轻量灵活

#### 1. 安装

> 这里使用`yum`安装

##### 1.1 添加docker yum源
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

##### 1.2 安装
```
$ yum install docker-ce
```

##### 1.3 启动docker
```
$ systemctl start docker

$ docker --version

$ docker info
```

##### 1.4 运行hello-world

```
$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

##### 1.5 代理设置

主机使用代理访问公网时，无法连接到公网需镜像仓库，解决办法是设置代理

```
$ vi /usr/lib/systemd/system/docker.service

[Service]

Environment="HTTP_PROXY=http://xxx" "HTTPS_PROXY=http://xxx"

$ systemctl restart docker
```

#### 2. 镜像管理

##### 2.1 查看镜像
```
docker images
```

> 等同于`docker image ls`

##### 2.2 镜像搜索
```
docker search centos
```

##### 2.3 镜像拉取
```
docker pull centos
```

##### 2.4 镜像删除
 
 ```
 docker image rm centos
 ```
 
 > 使用`-f`强制删除

#### 3. 镜像构建

##### 3.1 编写Dockerfile
```docker
FROM centos

CMD ["tail", "-f", "/dev/null"]
```

> `Docker`容器仅在它的`1号`进程（`PID`为`1`）运行时，会保持运行。如果`1号`进程退出了，`Docker`容器也就退出了。为了使容器保持运行状态，这里使用`tail -f /dev/null`命令启动`1号`进程并保持运行状态

##### 3.2 构建
```
docker build -t mycentos .
```

##### 3.3 查看构建的镜像
```
docker images
```

#### 4 运行镜像

##### 4.1 以守护进程运行
```
docker run -d mycentos 
```

##### 4.2 进入容器
```
docker exec -it mycentos bash
```

> 也可使用命令`docker run -it mycentos bash`直接运行并进入容器

##### 4.3 重启容器
```
docker restart ${CONTAINER ID}
```

##### 4.4 停止容器
```
docker stop ${CONTAINER ID}
```

> 支持`优雅退出`。先发送`SIGTERM`信号，在一段时间之后（`10s`）再发送`SIGKILL`信号。`Docker`内部的应用程序可以接收`SIGTERM`信号，然后做一些“退出前工作”，比如保存状态、处理当前请求等

##### 4.5 杀死容器进程
```
docker kill ${CONTAINER ID}
```

> 发送`SIGKILL`信号，应用程序直接退出

#### 5. Dockerfile
一个例子
```docker
# FROM 基础镜像
FROM centos

# 作者信息
MAINTAINER fool fool@aa.com

# 指定使用哪个用户执行RUN命令的用户
USER root

# copy本地目录到容器指定目录
COPY jdk1.8.0_60 /opt/jdk1.8.0_60

# 设置容器环境变量
ENV JAVA_HOME /opt/jdk1.8.0_60

# 添加文件到容器目录，对压缩文件会自动解压
ADD apache-tomcat-8.0.46.tar.gz /root

# 指定工作目录
WORKDIR /root

# 执行shell命令
RUN mv apache-tomcat-8.0.46 tomcat

# 暴露容器端口，在运行容器时通过参数`-p`指定宿主机到容器的端口映射，例如`docker run -p 8000:8080 -d tomcat`
EXPOSE 8080

# 挂载宿主机目录到容器
VOLUME ["/root"]

# 容器启动后执行的命令，只有最后一条有效，要求该命令保持运行状态。CMD和ENTRYPOINT的区别是前者可被docker run的参数覆盖
CMD ["/root/tomcat/bin/catalina.sh", "run"]
```

#### 参考文档

[Docker Documentation](https://docs.docker.com/)
[Install from a package](https://docs.docker.com/install/linux/docker-ce/centos/#install-from-a-package)