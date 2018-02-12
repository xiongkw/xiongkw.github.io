---
layout: post
title: 制作docker容器的tomcat基础镜像
categories: [编程, docker]
tags: [tomcat]
---

> 本文演示如何制作一个`tomcat`基础镜像并用其部署`war`应用

#### 1. 制作tomcat基础镜像

拉取`centos`基础镜像到本地：
```
docker pull 192.168.1.100:8080/public/centos:7

```

编写`dockerfile`,`vi Dockerfile`
```
# 基础镜像
FROM 192.168.1.100:8080/public/centos:7
# 作者信息
MAINTAINET xiongkw
# 安装jdk
ADD jdk1.7.0_79.tar.gz /usr/local
# 设置JAVA_HOME环境变量
ENV JAVA_HOME /usr/local/jdk1.7.0_79
# 安装tomcat
ADD apache-tomcat-8.0.46.tar.gz /usr/local

RUN chmod +x /usr/local/apache-tomcat-8.0.46/bin/*.sh
```
构建镜像：
```
docker build -t tomcat-centos:8.0.46 .
```

#### 2. 使用该镜像发布war包

编写`Dockerfile`,`vi Dockerfile`
```
FROM tomcat-centos:8.0.46
ADD myapp.war /usr/local/apache-tomcat-8.0.46/webapps

CMD ["/usr/local/apache-tomcat-8.0.46/bin/catalina.sh", "run"]
```
创建`docker`镜像：
```
docker build -t myapp.war:1.0.0 .
```
运行：
```
# 运行并进入容器命令行
docker run -it myapp.war:1.0.0

# 以后台进程运行
docker run -d myapp.war:1.0.0 

# 查看容器进程
docker ps|grep myapp

# 进入容器命令行：
docker exec -it ${id} bash
```
