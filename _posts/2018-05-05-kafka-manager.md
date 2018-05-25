---
layout: post
title: Kafka Manager试用
categories: [编程, java]
tags: [kafka, manager]
---


> 使用`Kafka`自带命令来配置和管理是一件非常头疼的事

#### 1. 简介

> 雅虎开源的Kafka集群管理工具

#### 2. 安装

##### 2.1 下载

到[kafka-manager](https://github.com/yahoo/kafka-manager)下载源码
```
git clone https://github.com/yahoo/kafka-manager.git
```

源码编译
```
cd kafka-manager

./sbt clean dist
```

> 最好在`linux`下编译，编译过程中会下载很多依赖包，并且很容易下载出错，需要多次重新编译

编译结果
```
target/universal/kafka-manager-1.3.3.17.zip
```

##### 2.2 安装

解压安装
```
unzip kafka-manager-1.3.3.17.zip
```

#### 3. 启动

修改配置
```
vi conf/application.conf
```
修改`zkhosts`
```
kafka-manager.zkhosts="127.0.0.1:8181"
```

> 注意这个`zkhosts`是用来做`kafka-manager`集群的，和`kafka`自身的`zk`无关

```
cd kafka-manager/bin

./kafka-manager -Dhttp.port=8080
```

#### 4. 访问页面
浏览器打开`http://192.168.1.100:8080`

第一次打开什么都没有，需要添加集群
[]({{site.url}}/public/images/2018-05-05-kafka-manager-01.png)

[]({{site.url}}/public/images/2018-05-05-kafka-manager-02.png)

> 这里的`zk`是`kafka`集群使用的`zk`

查看刚才添加的`kafka`集群
[]({{site.url}}/public/images/2018-05-05-kafka-manager-03.png)

查看`topics`
[]({{site.url}}/public/images/2018-05-05-kafka-manager-04.png)

查看`topic`详情
[]({{site.url}}/public/images/2018-05-05-kafka-manager-05.png)

#### 5. 参考文档

* [Kafka Manager](https://github.com/yahoo/kafka-manager)

