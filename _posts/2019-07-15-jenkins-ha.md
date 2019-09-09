---
layout: post
title: 通过JNLP部署jenkins-slave节点
categories: [编程, linux, docker]
tags: [jenkins, jnlp]
---

> 通过`JNLP`部署`jenkins-slave`节点

#### 1. 环境准备

```
192.168.1.101   master
192.168.1.102   slave
```

#### 2. 部署jenkins-master

##### 2.1 部署jenkins

编写`docker-compose.yml`

```
version: '2'

services:
  jenkins:
    container_name: jenkins
    restart: always
    image: jenkins/jenkins:2.184
    volumes:
      - /data/jenkins:/var/jenkins_home
    ports:
      - 8080:8080
      - 50000:50000
```

启动`master`

```
$ docker-compose up -d
```

##### 2.2 配置slave

![]({{site.url}}/public/images/2019-07-15-jenkins-ha-1.png)

![]({{site.url}}/public/images/2019-07-15-jenkins-ha-2.png)

> 通过`JNLP`方式安装`slave`时，需要填写以上的`name`和`secret`

#### 3. 部署slave

编写`docker-compose.yml`

```
version: '2'

services:
  jenkins-slave:
    container_name: jenkins-slave
    restart: always
    image: jenkins/jnlp-slave:3.29-1
    environment:
      JENKINS_URL: http://192.168.1.101:8080
      JENKINS_AGENT_NAME: slave
      JENKINS_SECRET: 874985f0b4145953ca7ca9138fb0fe1fd30a06505b52fbdcb9753932b2288a9c
      JENKINS_AGENT_WORKDIR: /home/jenkins/agent
```

> `JENKINS_AGENT_NAME`：`master`中配置的`slave`名称   
> `JENKINS_SECRET`: `master`中生成的`slave`节点密钥

启动`slave`

```
$ docker-compose up -d
```

#### 4. 查看slave状态

在`jenkins`控制台查看`slave`状态，略

#### 5. 参考

* [Jenkins JNLP Agent Docker image](https://github.com/jenkinsci/docker-jnlp-slave/)
* [jenkins/jenkins](https://hub.docker.com/r/jenkins/jenkins)

