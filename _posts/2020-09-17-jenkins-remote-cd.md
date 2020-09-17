---
layout: post
title: Jenkins分布式跨网络发布
categories: [编程, java]
tags: [jenkins]
---

> 大型分布式架构下，往往会跨资源池部署，而资源池的网络一般都是只能出不能进

#### 1. 架构

![]({{site.url}}/public/images/2020-09-17-jenkins-remote-cd_1.png)

#### 2. 部署

##### 2.1 Jenkins-master

下载`jenkins.war`直接启动即可

```
nohup java -DJENKINS_HOME=/data/jenkins -server -Xms4g -Xmx4g -jar jenkins.war --httpPort=8000 >/dev/null  2>&1 &
```

##### 2.1 Jenkins-agent

`Manage Jenkins - Manage Nodes - New Node` 增加一个远程节点

![]({{site.url}}/public/images/2020-09-17-jenkins-remote-cd_2.png)

查看节点详情
![]({{site.url}}/public/images/2020-09-17-jenkins-remote-cd_3.png)

下载`agent.jar`安装到远程主机并执行启动命令
```
java -jar agent.jar -jnlpUrl http://192.168.1.100:8000/computer/192.168.1.112/slave-agent.jnlp -secret xxx -workDir /data/jenkins
```

#### 3. 一个pipeline例子

创建一个`pipeline`，在指定标签(mac)的agent上运行，`sleep 3`的作用是执行完后等等3秒，否则可能获取不到输出日志

```
node ('mac'){
    stage("test") {
        sh "uname -a"
        sleep 3
    }
}
```

执行结果：

```
Started by user admin
Replayed #60
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] node
Running on 192.168.1.112 in /data/jenkins/workspace/test
[Pipeline] {
[Pipeline] stage
[Pipeline] { (test)
[Pipeline] sh
[Pipeline] sleep
Sleeping for 3 sec
+ uname -a
Darwin GBdeMac-mini.local 19.4.0 Darwin Kernel Version 19.4.0: Wed Mar  4 22:28:40 PST 2020; root:xnu-6153.101.6~15/RELEASE_X86_64 x86_64
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

#### 4. 关于代理

资源池内主机很可能需要经过代理才能访问`jenkins-master`，所以有时候需要给`agent`配置网络代理

```
-DsocksProxyHost=x.x.x.x -DsocksProxyPort=3000 -Dhttp.proxyHosts=x.x.x.x -Dhttp.proxyPort=3128
```

> 本文使用的`jenkins-2.176.1`需要同时配置`socks和http`代理（也可以使用全局代理软件，例如：`proxifier`）