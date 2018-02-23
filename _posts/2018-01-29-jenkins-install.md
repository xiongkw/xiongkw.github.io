---
layout: post
title: Jenkins安装
categories: [编程, jenkins]
tags: []
---


#### 1. 安装包下载

`jenkins`提供`linux、windows`等多种安装包，这里选择[war包下载](https://jenkins.io/download/)

## 2. 部署war包

```
mkdir jenkins-2.89.3

mv jenkins.war jenkins

tar zxvf jdk-8u60-linux-x64.tar.gz -C jenkins-2.89.3
```

> `jenkins-2.89.3`要求`1.8jdk`，为不影响系统已有`jdk`，这里直接解压到`jenkins`安装目录

## 3. 编写启动脚本

```
#!/bin/bash
JENKINS_HOME=/home/fool/jenkins-2.89.3
JENKINS_WEBROOT=$JENKINS_HOME/war
JENKINS_JAVA=$JENKINS_HOME/jdk1.8.0_60/bin/java
echo "Starting jeknins... jenkins home: ${JENKINS_HOME}, java: ${JENKINS_JAVA}"

nohup $JENKINS_JAVA -DJENKINS_HOME=$JENKINS_HOME -jar jenkins.war --httpPort=8000 --logfile=$JENKINS_HOME/logs/jenkins.log --webroot=$JENKINS_WEBROOT >logs/nohup.out 2>&1 &
echo $! > jenkins.pid
echo 'Jenkins started.'
```

## 4. 编写停止脚本

    #!/bin/bash

    echo 'Shutting down jenkins...'

    if [ -s "jenkins.pid" ]; then
            JENKINS_PID=`cat jenkins.pid`
            echo "killing jenkins pid [${JENKINS_PID}]"
            kill ${JENKINS_PID} && echo 'Jenkins stoped.' && rm jenkins.pid
    else
            exit 0
    fi

## 5. 启动

```
./startup.sh
```

## 6. 访问

在浏览器访问`http://192.168.1.100:8000`，需要输入密码

![]({{site.url}}/public/images/2018-01-29-jenkins-install-01.png)

填写`initAdminPassword`中的密码

```
cat secrets/initialAdminPassword
```

## 7. 代理设置

> 如果服务器需要通过代理连接公网，则需要进行如下配置，否则请忽略

![]({{site.url}}/public/images/2018-01-29-jenkins-install-02.png)

点击`Configure Proxy`按钮，设置`http`代理

![]({{site.url}}/public/images/2018-01-29-jenkins-install-03.png)

设置代理后发现依然提示`Offline`，网上找到解决方法：

```
vi hudson.model.UpdateCenter.xml

<?xml version='1.0' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>http://updates.jenkins.io/update-center.json</url>
  </site>
</sites>
```

> `https`改为`http`

删除配置文件后重启，为了能够重新进入初始化页面，这里需要 删除几个配置文件

```
./shutdown.sh

rm config.xml jenkins.CLI.xml nodeMonitors.xml

./startup.sh
```

进入初始化页面，重新填写初始密码，进入插件安装页面

## 8. 插件安装

这些选择建议安装就好，耐心等待插件安装完成

![]({{site.url}}/public/images/2018-01-29-jenkins-install-04.png)

可能会有安装失败的，例如：

![]({{site.url}}/public/images/2018-01-29-jenkins-install-05.png)

一般是由于网络原因导致，这里点击`Retry`按钮，直到插件安装完成，也可先忽略，过后再单独安装

## 9. 创建管理员账号

![]({{site.url}}/public/images/2018-01-29-jenkins-install-06.png)

安装完成

![]({{site.url}}/public/images/2018-01-29-jenkins-install-07.png)

## 10. slave安装

进入`系统管理-节点管理`页面

![]({{site.url}}/public/images/2018-01-29-jenkins-install-08.png)

新建节点

![]({{site.url}}/public/images/2018-01-29-jenkins-install-09.png)

进入节点配置页面

![]({{site.url}}/public/images/2018-01-29-jenkins-install-10.png)

新建的节点是离线状态，需要手动启动

![]({{site.url}}/public/images/2018-01-29-jenkins-install-11.png)

点击`Launch agent`启动节点

![]({{site.url}}/public/images/2018-01-29-jenkins-install-12.png)

启动后节点为在线状态

![]({{site.url}}/public/images/2018-01-29-jenkins-install-13.png)

## 附. 参考文档

[Installing Jenkins](https://jenkins.io/doc/book/installing/)