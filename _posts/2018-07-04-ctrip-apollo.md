---
layout: post
title: Ctrip-Apollo试用
categories: [编程, java, spring]
tags: [spring-cloud, apollo]
---


> 在微服务架构下，一个服务会发布到多个环境，例如开发、测试、生产等，而每个环境所对应的属性配置又不同，这样就需要有一个统一的配置管理中心来集中管理这些配置

#### 1. 简介

> `Apollo`（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景

组件介绍参考[Apollo配置中心设计](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1)

#### 3. 打包构建

> `Apollo`不提供部署安装包，需要自己下载源码构建

`Apollo`构建环境依赖`java`和`mysql`

* `Apollo`服务端：`1.8+`
* `Apollo`客户端：`1.7+`
* `mysql`：`5.6.5+`

##### 3.1 源码下载

从`github`下载源码

```
$ git clone https://github.com/ctripcorp/apollo.git
```

##### 3.2 修改构建脚本

`scripts/build.sh`

```properties
#apollo config db info
apollo_config_db_url=jdbc:mysql://localhost:3306/ApolloConfigDB?characterEncoding=utf8
apollo_config_db_username=用户名
apollo_config_db_password=密码（如果没有密码，留空即可）

# apollo portal db info
apollo_portal_db_url=jdbc:mysql://localhost:3306/ApolloPortalDB?characterEncoding=utf8
apollo_portal_db_username=用户名
apollo_portal_db_password=密码（如果没有密码，留空即可）

dev_meta=http://127.0.0.1:8001
fat_meta=http://127.0.0.1:8002
uat_meta=http://127.0.0.1:8003
pro_meta=http://127.0.0.1:8004
```

##### 3.3 构建

```
$ cd scripts

$ ./build.sh
```

#### 4. 服务端部署

##### 4.1 初始化数据库

```
source apollo/sql/apolloportaldb.sql

source apollo/sql/apolloconfigdb.sql

```

修改`portal`数据库`ApolloPortalDB.ServerConfig`

```
apollo.portal.envs  DEV,PRO
organizations   [{}]
...
```

修改`portal`数据库`ApolloConfigDB.ServerConfig`

```
eureka.service.url http://127.0.0.1:8081/eureka/,http://127.0.0.1:8082/eureka/
```

##### 4.2 部署configservice

解压`apollo-configservice-x.x.x-github.zip`，修改启动端口`scripts/startup.sh`

```
SERVER_PORT=8081
```

启动

```
$ cd scripts

$ ./startup.sh
```

##### 4.3 部署adminservice

解压`apollo-adminservice-x.x.x-github.zip`，修改启动端口`scripts/startup.sh`

```
SERVER_PORT=8082
```

启动

```
$ cd scripts

$ ./startup.sh
```

##### 4.4 部署portal

解压`apollo-portal-x.x.x-github.zip`，修改启动端口`scripts/startup.sh`

```
SERVER_PORT=8083
```

启动

```
$ cd scripts

$ ./startup.sh
```

##### 4.5 访问portal

```
http://127.0.0.1:8083
```

账号：`apollo/admin`

#### 5. 客户端接入

##### 5.1 服务端注册

在portal主页中创建应用`mytest`

##### 5.2 引入apollo-client依赖

`pom.xml`

```xml
<dependency>
    <groupId>com.ctrip.framework.apollo</groupId>
    <artifactId>apollo-client</artifactId>
    <version>0.11.0-SNAPSHOT</version>
</dependency>
```

##### 5.3 开启配置中心

```
@EnableApolloConfig
@SpringBootApplication
public class Application {

}
```

##### 5.4 配置属性注入

`apollo-client`提供`api`和`spring`注入的方式，这里直接使用`spring`属性注入

```
@Value("${mykey}")
private String key;
```

##### 5.5 启动服务

为了能够接入配置中心，服务启动时需要指定`应用ID`，`发布环境`等，这里使用`java系统属性`方式指定

java系统属性：
```
-Dapp.id=mytest -Ddev_meta=http://127.0.0.1:8081 -Denv=dev
```

运行`Application.main`启动服务，报错，异常如下：

```
Caused by: java.lang.IllegalArgumentException: Could not resolve placeholder 'mykey' in value "${mykey}"
```

原因是配置中心并没有定义这个属性

##### 5.6 定义配置

登录`portal`，在`DEV`环境`application`Namespace下新增配置`mykey: aaa`并发布

再次启动服务，成功。访问`mytest`服务，发现`mykey`的值为`aaa`

##### 5.7 配置更新

修改`mykey`值为`bbb`并发布，访问`mytest`服务，发现`mykey`的值变为了`bbb`

#### 6. 踩坑集合

##### 6.1 源码构建依赖运行时环境

例如需要在`build.sh`中指定`jdbc_`和`meta server`，对不同环境需要多次打包`config-service`和`admin-service`

##### 6.2 配置不统一

例如`Config Service`和`Admin Service`的`eureka.service.url`配置在数据库中，而`Portal Service`和`client`的`meta server`配置在属性文件中(apollo-env.properties)

##### 6.3 集群的情况下需要借助nginx做负载均衡

例如多个`meta server`的情况

##### 6.4 环境自定义问题

环境硬编码在`com.ctrip.framework.apollo.core.enums.Env`中，扩展性差

##### 6.5 配置文件打在jar包里

例如`apollo-core/apollo-env.properties`

##### 6.6 开放接口不全

缺少应用注册接口，只提供了供管理员操作的应用注册页面(`http://{portal_address}/open/manage.html`)

#### 7. 参考

* [Apollo（配置中心）](https://github.com/ctripcorp/apollo)

